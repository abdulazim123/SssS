#!/bin/sh
# Vault dev-mode setup — seeds secrets for local development.
# In production, secrets are managed by your Vault operators / Terraform.

export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=dev-root-token

echo "[vault-init] Enabling KV v2..."
vault secrets enable -path=secret kv-v2 2>/dev/null || true

echo "[vault-init] Writing dev secrets..."
vault kv put secret/saas \
  jwt_secret="dev-jwt-secret-change-in-production-min32!" \
  db_password="changeme_local" \
  db_user="saas_user" \
  redis_password="changeme_local"

echo "[vault-init] Enabling AppRole auth..."
vault auth enable approle 2>/dev/null || true

echo "[vault-init] Creating policy..."
vault policy write saas-app - <<EOF
path "secret/data/saas" {
  capabilities = ["read"]
}
path "secret/metadata/saas" {
  capabilities = ["read"]
}
EOF

echo "[vault-init] Creating AppRole..."
vault write auth/approle/role/saas-app \
  token_policies="saas-app" \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=24h

echo "[vault-init] Fetching role credentials..."
ROLE_ID=$(vault read -field=role_id auth/approle/role/saas-app/role-id)
SECRET_ID=$(vault write -f -field=secret_id auth/approle/role/saas-app/secret-id)

echo "[vault-init] ─────────────────────────────────────────"
echo "[vault-init] VAULT_ROLE_ID=${ROLE_ID}"
echo "[vault-init] VAULT_SECRET_ID=${SECRET_ID}"
echo "[vault-init] Add these to your .env or docker-compose env"
echo "[vault-init] ─────────────────────────────────────────"
echo "[vault-init] Done."
# SssS
