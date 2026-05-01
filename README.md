SaaS Backend — Security-First Node.js API
A production-ready Express backend for a multi-tenant SaaS application. Secrets are never stored in environment variables or code — they are fetched at runtime from HashiCorp Vault or AWS Secrets Manager, held in memory, and rotated automatically.
---
Architecture overview
```
Request → CDN / WAF
        → Rate limiter (express-rate-limit)
        → API Gateway (Express + Helmet)
        → Auth middleware (JWT from SecretsManager)
        → RBAC + Tenant isolation middleware
        → Route handler
        → DB (pg pool, credentials from SecretsManager)
           Redis (credentials from SecretsManager)
        → Audit logger (immutable trail)
        → Response
```
---
Secrets flow
```
App starts
  └─ SecretsManager.getInstance()
       ├─ Vault: AppRole login → short-lived token → read secret/data/saas
       └─ AWS:   IAM role → GetSecretValue(ARN)

       └─ Cache in memory { value, expiresAt }
            └─ Schedule re-fetch at 80% of TTL
                 └─ On re-fetch: update cache, no restart needed
```
Secrets fetched:
`jwt_secret` — used to sign/verify all JWTs
`db_password` — injected into pg.Pool at creation
`redis_password` — injected into Redis client at creation
---
Project structure
```
src/
├── server.js                  Entry point, graceful shutdown
├── app.js                     Express setup, middleware chain
├── config/
│   └── schema.sql             Postgres schema + RLS policies
├── services/
│   ├── secrets.service.js     Vault / AWS SM abstraction
│   ├── auth.service.js        JWT, bcrypt, user management
│   ├── db.service.js          pg.Pool with secret-injected creds
│   └── cache.service.js       Redis with secret-injected password
├── middleware/
│   └── security.middleware.js authenticate, requireRole, enforceTenant, rate limits
├── routes/
│   ├── auth.routes.js         /auth/register, login, refresh, logout, me
│   └── api.routes.js          /api/v1/projects, members, audit-log
└── utils/
    └── logger.js              Winston + audit logger, auto-redacts secrets
tests/
    └── auth.test.js           Jest + Supertest
```
---
Getting started
Prerequisites
Node.js 20+
Docker + Docker Compose
1. Clone and install
```bash
cp .env.example .env
npm install
```
2. Start dependencies
```bash
docker-compose up postgres redis vault
```
Vault will print `VAULT_ROLE_ID` and `VAULT_SECRET_ID` in its logs. Add them to your `.env`.
3. Run the server
```bash
npm run dev
```
4. Test
```bash
# Register
curl -X POST http://localhost:3000/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"me@example.com","password":"SecurePass123!","tenantId":"00000000-0000-0000-0000-000000000001"}'

# Login
curl -X POST http://localhost:3000/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"me@example.com","password":"SecurePass123!"}'

# Use token
curl http://localhost:3000/api/v1/projects \
  -H 'Authorization: Bearer <accessToken>'
```
---
Security design decisions
Concern	Decision
Secrets in env vars	Never. Fetched from Vault / AWS SM at runtime.
Password storage	bcrypt, 12 rounds
JWT secret	Fetched from SecretsManager; rotated without restart
Token revocation	JTI blocklist in Redis
Tenant isolation	Enforced in middleware AND Postgres RLS (two layers)
Audit log	Immutable (Postgres rules block UPDATE/DELETE)
Rate limiting	Global (per user/IP) + strict auth limiter
Timing attacks	Constant-time password compare; dummy hash on unknown email
Error leakage	Stack traces stripped in production responses
Logging	Automatic redaction of password/token/secret keys
CORS	Allowlist only; credentials mode
HTTP headers	Helmet with strict CSP + HSTS preload
---
Switching secrets backends
Set `SECRETS_BACKEND` in your environment:
Value	When to use
`vault`	Self-hosted or HCP Vault
`aws`	AWS ECS / Lambda / EC2 with IAM role
`env`	Local development only — reads from `process.env`
---
Production checklist
[ ] `SECRETS_BACKEND=vault` or `aws` (never `env`)
[ ] Vault AppRole `secret_id` injected by CI/CD, not stored anywhere
[ ] Postgres RLS enabled (schema.sql already sets this up)
[ ] `DB_SSL=true` with a valid CA cert
[ ] Redis TLS enabled
[ ] `AUDIT_LOG_ENABLED=true`
[ ] Log aggregation pipeline pointed at `logs/audit.log`
[ ] Alert on `audit_events` rows with `outcome = 'failure'`
[ ] JWT `JWT_EXPIRES_IN=15m`, refresh tokens stored server-side
[ ] Run as non-root Docker user (Dockerfile already does this)
[ ] `ALLOWED_ORIGINS` set to your actual frontend domain
