# Lecture 6 - Authorization (Keycloak Authz + OpenFGA)

Two fine-grained authorization PDPs plus one demo backend:

- `infra/keycloak` - the `api-security` realm defines roles
  (`document-viewer/editor/admin`), users `alice`/`bob`, and a confidential
  client `documents-api` with **Authorization Services** (resource `Document`,
  scopes view/edit/delete, role-based scope permissions).
- `infra/openfga` - an **OpenFGA** server (Zanzibar/ReBAC) on Postgres.
- `applications/authz-demo` - a Spring Boot service exposing the same documents
  API two ways: `/kc/**` decided by Keycloak, `/fga/**` decided by OpenFGA.

## Backing services (already provisioned)

Both components use the shared Postgres on `192.168.50.10:5432` (same instance
as `keycloak` and `spa_token_demo`) and read their DB credentials from Vault via
ExternalSecrets. The following were created during setup:

- Postgres databases + roles: `openfga` and `authz_demo` (passwords are random,
  stored only in Vault and Postgres - not in git).
- Vault KV (v2, mount `secret/`):
  - `secret/api-security/openfga-db`   - `username`, `password`
  - `secret/api-security/authz-demo-db` - `username`, `password`

No Keycloak client secret is stored: the backend asks Keycloak for decisions
using the UMA entitlement flow (`response_mode=decision`) with the caller's own
access token.

### Reproducing the backing services (if rebuilding the lab)

```bash
# 1. Postgres (run on the haproxy VM that hosts Postgres):
#    multipass exec haproxy -- sudo -u postgres psql
CREATE ROLE openfga    LOGIN PASSWORD '<generated>';
CREATE DATABASE openfga    OWNER openfga;
CREATE ROLE authz_demo LOGIN PASSWORD '<generated>';
CREATE DATABASE authz_demo OWNER authz_demo;

# 2. Vault (write the same credentials ESO reads):
vault kv put secret/api-security/openfga-db    username=openfga    password='<generated>'
vault kv put secret/api-security/authz-demo-db username=authz_demo password='<generated>'

# 3. Force ESO to resync if the ExternalSecrets synced before the Vault writes:
kubectl -n openfga      annotate externalsecret openfga-db-credentials force-sync="$(date +%s)" --overwrite
kubectl -n applications annotate externalsecret authz-demo-db         force-sync="$(date +%s)" --overwrite
```

> Note on the realm: Keycloak's `--import-realm` uses `IGNORE_EXISTING`, so it
> only imports the realm when it is absent. After changing
> `realm-configmap.yaml` for an existing realm, delete the realm
> (`DELETE /admin/realms/api-security`) and restart the Keycloak deployment, or
> apply the change via the Admin API. A fresh import recreates everything,
> including the `documents-api` authorization config.

## Try it

```bash
KC=https://keycloak.192.168.50.10.nip.io/realms/api-security
get() { curl -sk -X POST $KC/protocol/openid-connect/token \
  -d grant_type=password -d client_id=spa-token-demo -d username=$1 -d password=$1 \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])'; }
ALICE=$(get alice); BOB=$(get bob)
BASE=https://authz-demo.192.168.50.10.nip.io

# Keycloak side (role/scope): bob may view, may not delete
curl -sk -H "Authorization: Bearer $BOB" $BASE/kc/documents              # 200
curl -sk -X DELETE -H "Authorization: Bearer $BOB" $BASE/kc/documents/1  # 403

# OpenFGA side (per-object): bob sees doc 1 (via team) + doc 2 (owner),
# but NOT doc 3 - changing the id does not help. BOLA prevented.
curl -sk -H "Authorization: Bearer $BOB" $BASE/fga/documents             # [1,2]
curl -sk -H "Authorization: Bearer $BOB" $BASE/fga/documents/3           # 403
```
