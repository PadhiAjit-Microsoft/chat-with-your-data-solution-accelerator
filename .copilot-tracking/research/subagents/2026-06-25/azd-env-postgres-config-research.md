<!-- markdownlint-disable-file -->
# Research: azd env + PostgreSQL/pgvector deploy config (exact redeploy runbook inputs)

Status: Complete

Goal: Document the azd environment under v2/.azure and the PostgreSQL / pgvector
deployment configuration so an exact "code-only redeploy against EXISTING
resources" runbook can be produced.

All real subscription/tenant/principal IDs, resource-group names, and resource
suffixes are REDACTED to placeholders per repo Hard Rule #18 and the user
directive. Only KEY NAMES and non-secret structural facts are recorded.

---

## 1. azd environment — v2/.azure

Folder listing of v2/.azure:

- `config.json` (v2/.azure/config.json) — `{"version":1,"defaultEnvironment":"<AZD_ENV_NAME>"}`. There is exactly ONE azd env and it is the default.
- `<AZD_ENV_NAME>/` — the single env folder. Contains `.env`, `.env.lock`, `config.json`.
- `.gitignore`, `.state-change` — azd housekeeping.

Per-env files:

- v2/.azure/`<AZD_ENV_NAME>`/config.json — empty object `{}` (no per-env overrides).
- v2/.azure/`<AZD_ENV_NAME>`/.env — the live env-var snapshot azd reads for `azd deploy`.

### Env name to pass with `-e`

- The azd env name is the single subfolder under v2/.azure and the `defaultEnvironment` in v2/.azure/config.json. Refer to it as `<AZD_ENV_NAME>`.
- Because it is already the default, `-e <AZD_ENV_NAME>` is OPTIONAL on `azd deploy`, but passing it explicitly is safest in scripts.

### Database / index keys in .env (NAMES only; DB-relevant values are non-secret discriminators)

These confirm PostgreSQL + pgvector are the ACTIVE mode (values shown are
closed-set discriminators, not secrets):

- `AZURE_DB_TYPE` = `postgresql`
- `AZURE_ENV_DATABASE_TYPE` = `postgresql`  (the azd-prompt answer that drives the Bicep `databaseType` param)
- `AZURE_INDEX_STORE` = `pgvector`
- `AZURE_POSTGRES_HOST` = `psql-<SUFFIX>.postgres.database.azure.com`
- `AZURE_POSTGRES_NAME` = `psql-<SUFFIX>`
- `AZURE_POSTGRES_ENDPOINT` = `postgresql://psql-<SUFFIX>.postgres.database.azure.com:5432/cwyd?sslmode=require`
- `AZURE_POSTGRES_ADMIN_PRINCIPAL_NAME` = `<AZURE_PRINCIPAL_UPN>`
- `AZURE_ENV_POSTGRES_ADMIN_PRINCIPAL_NAME` = `<AZURE_PRINCIPAL_UPN>`

Cosmos / Search keys are present but EMPTY (proves cosmos/search NOT deployed):

- `AZURE_AI_SEARCH_ENDPOINT` = `""`
- `AZURE_AI_SEARCH_NAME` = `""`
- `AZURE_COSMOS_ACCOUNT_NAME` = `""`
- `AZURE_COSMOS_ENDPOINT` = `""`

WAF / mode flags in .env (all default-off):

- `AZURE_ENV_ENABLE_MONITORING` = `false`
- `AZURE_ENV_ENABLE_SCALABILITY` = `false`
- `AZURE_ENV_ENABLE_REDUNDANCY` = `false`
- `AZURE_ENV_ENABLE_PRIVATE_NETWORKING` = `false`

### Other .env key NAMES present (env-specific — values redacted)

Env-identity / resource-name keys (all env-specific):
`AZURE_ENV_NAME` (=`<AZD_ENV_NAME>`), `AZURE_SOLUTION_SUFFIX` (=`<SUFFIX>`),
`AZURE_RESOURCE_GROUP` (=`<RESOURCE_GROUP>`), `AZURE_SUBSCRIPTION_ID`,
`AZURE_TENANT_ID`, `AZURE_LOCATION` (non-AI region),
`AZURE_ENV_AI_SERVICE_LOCATION` / `AZURE_AI_SERVICE_LOCATION` (model region),
`AZURE_UAMI_CLIENT_ID`, `AZURE_UAMI_PRINCIPAL_ID`, `AZURE_UAMI_RESOURCE_ID`.

Service / endpoint keys (env-specific resource names embed `<SUFFIX>`):
`AZURE_BACKEND_URL` (`https://ca-backend-<SUFFIX>.<aca-env-domain>.azurecontainerapps.io`),
`AZURE_FRONTEND_URL` (`https://app-frontend-<SUFFIX>.azurewebsites.net`),
`AZURE_FUNCTION_APP_NAME` (`func-<SUFFIX>`),
`AZURE_FUNCTION_APP_URL` (`https://func-<SUFFIX>.azurewebsites.net`),
`AZURE_CONTAINER_REGISTRY_NAME` (`cr<SUFFIX>`),
`AZURE_CONTAINER_REGISTRY_ENDPOINT` (`cr<SUFFIX>.azurecr.io`),
`AZURE_STORAGE_ACCOUNT_NAME` (`st<SUFFIX>`),
`AZURE_STORAGE_BLOB_ENDPOINT`, `AZURE_AI_PROJECT_ENDPOINT`,
`AZURE_AI_SERVICES_ENDPOINT`, `AZURE_OPENAI_ENDPOINT`,
`AZURE_CONTENT_SAFETY_ENDPOINT` / `AZURE_CONTENT_SAFETY_NAME`,
`AZURE_SPEECH_SERVICE_NAME` / `AZURE_SPEECH_ACCOUNT_RESOURCE_ID` / `AZURE_SPEECH_SERVICE_REGION`,
`AZURE_DOCUMENTS_CONTAINER` (=`documents`), `AZURE_DOC_PROCESSING_QUEUE` (=`doc-processing`).

Model / version keys (non-secret config):
`AZURE_OPENAI_GPT_DEPLOYMENT` (=`gpt-5.1`),
`AZURE_OPENAI_REASONING_DEPLOYMENT` (=`o4-mini`),
`AZURE_OPENAI_EMBEDDING_DEPLOYMENT` (=`text-embedding-3-large`),
`AZURE_OPENAI_API_VERSION`, `AZURE_AI_AGENT_API_VERSION`,
`AZURE_AI_SEARCH_KNOWLEDGE_BASE_NAME` (=`cwyd-kb`),
`AZURE_AI_SEARCH_KNOWLEDGE_SOURCE_NAME` (=`cwyd-index-ks`),
`AZURE_AI_SEARCH_KNOWLEDGE_BASE_API_VERSION`.

azd build-state keys:
`SERVICE_BACKEND_IMAGE_NAME` (last backend image tag `…/backend-<AZD_ENV_NAME>:azd-deploy-<epoch>`),
`SERVICE_BACKEND_RESOURCE_EXISTS` (=`true`).

VNet keys present but EMPTY (private networking off):
`AZURE_VNET_NAME`, `AZURE_VNET_RESOURCE_ID`, `AZURE_BASTION_NAME`,
`AZURE_APP_INSIGHTS_CONNECTION_STRING`.

---

## 2. azure.yaml — services → Azure resource mapping

File: v2/azure.yaml (name: `chat-with-your-data-v2`).

`infra:` block → provider `bicep`, path `infra`, module `main` (v2/azure.yaml:29-32).

### Service-name → host mapping table (these are the `azd deploy <service>` names)

| `azd deploy` service | project path        | host         | language | Azure resource (tag in main.bicep)        | image / build path                                   | azure.yaml lines |
|----------------------|---------------------|--------------|----------|-------------------------------------------|------------------------------------------------------|------------------|
| `backend`            | ./src/backend       | containerapp | py       | Container App `ca-backend-<SUFFIX>` (tag `backend`)  | docker: ../../docker/Dockerfile.backend, context ../.., remoteBuild: true | v2/azure.yaml:104-114 |
| `frontend`           | ./src/frontend      | appservice   | js       | App Service `app-frontend-<SUFFIX>` (tag `frontend`) | build-from-source; dist ./build-output; prepackage hook `package-frontend.{sh,ps1}` | v2/azure.yaml:115-136 |
| `function`           | ./build-functions   | function     | py       | Function App `func-<SUFFIX>` (tag `function`)        | zip deploy of build-functions/; prepackage hook `prepackage-function.{sh,ps1}` | v2/azure.yaml:137-157 |

Notes:
- The three deployable service names are exactly: **`backend`**, **`frontend`**, **`function`**.
- `frontend` host is `appservice` with NO `docker:` block (the docker block on an appservice host was the root cause of BUG-0081; current azure.yaml uses build-from-source + a prepackage staging hook into ./build-output). See v2/azure.yaml:115-136.
- `function` project path is `./build-functions` (the generated deploy artifact), NOT `./src/functions`. Its prepackage hook regenerates build-functions/ from src/ and is SERVICE-scoped (`services.function.hooks.prepackage`) so it fires on a targeted `azd deploy function` (BUG-0058 fix). See v2/azure.yaml:137-157.
- Project-level hooks: `postprovision` → scripts/post-provision.{sh,ps1} (creates pgvector extension in postgresql mode; interactive), `postdeploy` → scripts/upload-sample-data.{sh,ps1} (seeds sample docs; continueOnError true). See v2/azure.yaml:206-228.

---

## 3. main.bicep / main.parameters.json — PostgreSQL + pgvector + orchestrator

### The single mode-selecting parameter

- `databaseType` (v2/infra/main.bicep:86) — `param databaseType string = 'cosmosdb'`; allowed values `cosmosdb` | `postgresql` (v2/infra/main.bicep:82-85). Selects BOTH chat-history backend AND vector index store. `postgresql` ⇒ PostgreSQL Flexible Server with pgvector; Azure AI Search is NOT deployed.
- Bound in v2/infra/main.parameters.json:18-20 → `"databaseType": { "value": "${AZURE_ENV_DATABASE_TYPE=cosmosdb}" }`. The env sets `AZURE_ENV_DATABASE_TYPE=postgresql`, so the active value is **postgresql**.
- Surfaced as the azd typed-prompt `databaseType` in v2/azure.yaml:39-49 (default cosmosdb; persisted to `AZURE_ENV_DATABASE_TYPE`).

### PostgreSQL Flexible Server module (deployed ONLY when databaseType==postgresql)

- v2/infra/main.bicep:1488 — `module postgresServer 'br/public:avm/res/db-for-postgre-sql/flexible-server:0.15.3' = if (databaseType == 'postgresql')`.
- Server name `psql-<SUFFIX>`; version `16`; storage 32 GB; SKU `Standard_B2s` / Burstable (scalability off) (v2/infra/main.bicep:1490-1499).
- Auth is Entra-only: `activeDirectoryAuth: 'Enabled'`, `passwordAuth: 'Disabled'` (v2/infra/main.bicep:1503-1507).
- Administrators: the workload UAMI `id-<SUFFIX>` (ServicePrincipal) always, plus the deployer principal when `postgresAdminPrincipalId` is set (v2/infra/main.bicep:1508-1525).
- Database created: `cwyd` (UTF8) (v2/infra/main.bicep:1527-1533).
- **pgvector enabled** via server configuration `azure.extensions = VECTOR` (`source: user-override`) (v2/infra/main.bicep:1534-1540). The CREATE EXTENSION is run by the postprovision hook scripts/post-provision.py.

### PostgreSQL-related parameters (main.bicep / main.parameters.json)

- `postgresAdminPrincipalId` (v2/infra/main.bicep:1462, default `''`) ← main.parameters.json:86-88 `${AZURE_ENV_POSTGRES_ADMIN_PRINCIPAL_ID=${AZURE_PRINCIPAL_ID}}`.
- `postgresAdminPrincipalName` (v2/infra/main.bicep:1465, default `''`, REQUIRED when postgresql) ← main.parameters.json:89-91 `${AZURE_ENV_POSTGRES_ADMIN_PRINCIPAL_NAME}`. Fail-fast guard if empty in postgresql mode (v2/infra/main.bicep:1467-1483).
- `postgresAdminPrincipalType` (v2/infra/main.bicep:1486, default User/ServicePrincipal) ← main.parameters.json:92-94 `${AZURE_ENV_POSTGRES_ADMIN_PRINCIPAL_TYPE=User}`.

### Index store value (derived, not a standalone param)

- v2/infra/main.bicep:1623 — `var indexStoreValue = databaseType == 'cosmosdb' ? 'AzureSearch' : 'pgvector'`.
- Wired into BOTH runtimes as env `AZURE_INDEX_STORE`:
  - backend Container App: v2/infra/main.bicep:1847 `{ name: 'AZURE_INDEX_STORE', value: indexStoreValue }` (and `AZURE_DB_TYPE` at :1846).
  - Function App: v2/infra/main.bicep:2201 `{ name: 'AZURE_INDEX_STORE', value: indexStoreValue }` (and `AZURE_DB_TYPE` at :2200).
- Bicep output `AZURE_INDEX_STORE` (v2/infra/main.bicep:2467) and `AZURE_DB_TYPE` (v2/infra/main.bicep:2464).

### Orchestrator toggle — IMPORTANT (postgresql ⇒ langgraph, NOT agent_framework)

- v2/infra/main.bicep:1884 — `{ name: 'CWYD_ORCHESTRATOR_NAME', value: databaseType == 'postgresql' ? 'langgraph' : 'agent_framework' }`.
- The default orchestrator is a runtime env var `CWYD_ORCHESTRATOR_NAME` (OrchestratorSettings env_prefix `CWYD_ORCHESTRATOR_` + field `name`; v2/infra/main.bicep:1880-1884).
- Therefore, in the ACTIVE postgresql deployment, the default orchestrator the backend boots with is **`langgraph`**, not `agent_framework`. (Both orchestrators register and are runtime-switchable per request; ADR 0027 also allows agent_framework app-side RAG on pgvector — but the Bicep DEFAULT for postgresql mode is langgraph.) `CWYD_ORCHESTRATOR_NAME` is NOT present in the .env snapshot because it is a Bicep-injected container env var, not an azd env-file var.

### Other deploy-relevant params (main.parameters.json)

- `solutionName` ← `${AZURE_ENV_SOLUTION_NAME=${AZURE_ENV_NAME}}`; `location` ← `${AZURE_LOCATION}`; `azureAiServiceLocation` ← `${AZURE_ENV_AI_SERVICE_LOCATION=…}`.
- `ingestionTrigger` ← `${AZURE_ENV_INGESTION_TRIGGER=direct_enqueue}` (main.parameters.json:21-23). Currently `direct_enqueue` (Event Grid cutover deferred per bugs.md BUG-0054).
- WAF flags `enableMonitoring` / `enableScalability` / `enableRedundancy` / `enablePrivateNetworking` all `=false`.
- `backendContainerRegistryHostname` ← `${AZURE_CONTAINER_REGISTRY_ENDPOINT=}`; `backendContainerImageTag` ← `${AZURE_ENV_IMAGE_TAG=latest}`.

---

## 4. Redeploy commands documented in the repo

Makefile: v2/Makefile defines ONLY `typecheck` / `test` / `lint` (no azd/deploy targets). No deploy command lives in the Makefile.

README: v2/README.md documents the `cd v2` convention (v2/README.md:47, :65) for the test lanes; it has no dedicated `azd deploy` runbook section.

Canonical `azd deploy` commands (from azure.yaml comments + worklogs + bugs.md):

- Per-service code-only redeploy (the common case against existing resources):
  - `azd deploy backend`
  - `azd deploy frontend`
  - `azd deploy function`
- All services: `azd deploy --all` (used in worklog 2026-06-16).
- Function deploy with extended wait (Flex Consumption remote-build is slow):
  - `azd deploy function --timeout 2400` (worklog 2026-06-23:95, bugs.md:981)
  - `azd deploy function --timeout 1800` (also used)
- Non-interactive: `azd deploy function --no-prompt` (worklog 2026-06-20:38).
- Infra refresh (re-applies Bicep-derived env): `azd provision` (re-applies env vars; note worklog warnings that `--set-env-vars` live overrides are wiped by the next `azd provision`/`azd deploy`).
- Full cold path: `azd up` = `azd provision` + `azd deploy` + hooks (project_status.md:412).

Function-specific prepackage caveat (BUG-0058, now fixed via service-scoped hook):
- If shipping the function and unsure the hook fired, regenerate the artifact first: `uv run python scripts/prepackage_function.py`, then `azd deploy function`. The service-scoped prepackage hook (v2/azure.yaml:137-157) now does this automatically on `azd deploy function`.

---

## 5. azd cwd gotcha — MUST run from v2/

Confirmed: there are TWO azure.yaml files, so azd resolves a DIFFERENT project depending on the shell's working directory.

- Repo-root azure.yaml: `c:\workstation\Microsoft\github\cwyd-pg\azure.yaml` — this is the **v1** project (`name: chat-with-your-data-solution-accelerator`, template `…@1.7.0`; services `web`, `backend`, `adminweb`, `function` all `host: appservice`/`function`). Running `azd deploy` from the repo root targets V1.
- v2 azure.yaml: `c:\workstation\Microsoft\github\cwyd-pg\v2\azure.yaml` — the **v2** project (`name: chat-with-your-data-v2`; services `backend`/containerapp, `frontend`/appservice, `function`/function).

Therefore every v2 `azd …` command MUST be run with shell cwd = `v2/`:

- worklog 2026-06-24:123 — "Backend redeployed with `azd deploy backend` (run from `v2/`, ~2.5 min)".
- README convention is `cd v2` before any v2 command (v2/README.md:47, :65).
- The session terminals show the established pattern: `Set-Location c:\workstation\Microsoft\github\cwyd-pg\v2; azd deploy function …`.

PowerShell form: `Set-Location c:\workstation\Microsoft\github\cwyd-pg\v2` (or `cd v2`) BEFORE `azd deploy <service>`.

---

## 6. Exact redeploy command syntax (code-only, EXISTING resources)

PowerShell, run from the repo root (each block self-contained):

```powershell
# Backend (Container App) — ACR remote build + revision swap, ~2.5 min
Set-Location c:\workstation\Microsoft\github\cwyd-pg\v2
azd deploy backend -e <AZD_ENV_NAME> --no-prompt

# Frontend (App Service) — build-from-source via prepackage hook
Set-Location c:\workstation\Microsoft\github\cwyd-pg\v2
azd deploy frontend -e <AZD_ENV_NAME> --no-prompt

# Function (Flex Consumption) — slow remote build; raise the wait timeout
Set-Location c:\workstation\Microsoft\github\cwyd-pg\v2
azd deploy function -e <AZD_ENV_NAME> --no-prompt --timeout 2400
```

- `-e <AZD_ENV_NAME>` is optional (it is the default env) but explicit is safest.
- `--no-prompt` runs unattended (no interactive confirmations).
- `--timeout 2400` is recommended for `function` only (Flex remote-build slowness, BUG-0058/0062 class). Backend/frontend complete in minutes.
- The known Function storage-firewall gotcha (BUG-0062 / worklog 2026-06-25): if the package upload returns `403 InaccessibleStorageException`, temporarily open the storage account public network, deploy, then re-lock — `az storage account update -n st<SUFFIX> -g <RESOURCE_GROUP> --public-network-access Enabled --default-action Allow`, run `azd deploy function`, then re-lock with `--public-network-access Disabled`. (This only affects `function`; backend/frontend do not touch that account.)

---

## Key discoveries (summary)

1. Single azd env `<AZD_ENV_NAME>` (default) under v2/.azure; per-env config.json is empty; .env confirms `AZURE_DB_TYPE=postgresql`, `AZURE_INDEX_STORE=pgvector`, Cosmos/Search keys empty.
2. Three deployable azd services: `backend` (containerapp), `frontend` (appservice), `function` (function).
3. `databaseType=postgresql` (via `AZURE_ENV_DATABASE_TYPE`) is the single switch; it deploys AVM flexible-server 0.15.3 (`psql-<SUFFIX>`, v16, db `cwyd`, Entra-only) with pgvector via `azure.extensions=VECTOR`. `indexStoreValue` derives `pgvector`.
4. Default orchestrator in postgresql mode is **langgraph** (`CWYD_ORCHESTRATOR_NAME = databaseType=='postgresql' ? 'langgraph' : 'agent_framework'`, main.bicep:1884) — NOT agent_framework. Both register and are runtime-switchable.
5. MUST run `azd` from `v2/` — repo root has a competing v1 azure.yaml.

## Clarifying questions

- The user's goal phrasing assumed the active orchestrator is `agent_framework`, but the deployed postgresql-mode DEFAULT is `langgraph` (main.bicep:1884). Confirm which orchestrator the redeploy runbook should document/assume as the default. (If agent_framework is desired on pgvector, it is supported per ADR 0027 but would require setting `CWYD_ORCHESTRATOR_NAME=agent_framework`, which is NOT the current Bicep default.)
- The runbook target is code-only `azd deploy` against EXISTING resources (no `azd provision`). Confirm whether `frontend` (BUG-0081 history) and `function` (BUG-0058 prepackage, BUG-0062 storage firewall) are in scope, or only `backend`.

## Recommended next research (not done here)

- [ ] Read scripts/post-provision.py to capture the exact pgvector CREATE EXTENSION + schema-init steps (only needed if a `azd provision`/postprovision re-run is in scope).
- [ ] Read scripts/package-frontend.ps1 + scripts/prepackage-function.ps1 to document the staging contracts if frontend/function redeploys are in scope.
- [ ] Confirm current live `CWYD_ORCHESTRATOR_NAME` on `ca-backend-<SUFFIX>` (az containerapp show) vs the Bicep default, in case a live `--set-env-vars` override differs.
