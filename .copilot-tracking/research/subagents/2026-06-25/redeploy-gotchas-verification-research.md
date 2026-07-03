<!-- markdownlint-disable-file -->
# CWYD v2 Redeploy — Gotchas, Pre-Checks & Post-Deploy Verification

Research date: 2026-06-25
Scope: Safe redeploy of CWYD **v2** to its existing Azure environment running **PostgreSQL** (pgvector index store).
Method: read-only over repo files only (no live Azure calls). All Azure identifiers shown as placeholders.

## Research questions

1. Documented redeploy gotchas from the 2026-06-24 / 2026-06-25 worklogs (azd cwd requirement, per-service deploy patterns, revision restart).
2. Full details of BUG-0082, BUG-0081, BUG-0058, BUG-0055, BUG-0054 — symptom, root cause, status, redeploy pre-check implied.
3. Backend FastAPI lifespan / startup DB connection path — is there a bounded connect timeout / fail-fast? (BUG-0082)
4. Post-deploy verification approach — health endpoints and what a healthy `/api/admin/status` returns.
5. Pre-check that Postgres Flexible Server must be `Ready` (not `Stopped`) before redeploying the backend, with the az CLI commands.

---

## Pre-Checks (run BEFORE any redeploy)

Ordered, do these in sequence before touching `azd`:

1. **PRE-1 — Start the PostgreSQL Flexible Server (CRITICAL for backend).** The Burstable-tier `psql-<SUFFIX>` auto-stops after ~7 idle days. A stopped server still answers TCP but resets the TLS handshake, and the backend lifespan has **no connect timeout** (BUG-0082), so deploying the backend against a stopped DB produces a permanent crash-loop. Always confirm `Ready` first:
   - Check: `az postgres flexible-server show -g <RESOURCE_GROUP> -n <PG_NAME> --query state -o tsv` (expect `Ready`; `Stopped` = must start).
   - Start: `az postgres flexible-server start -g <RESOURCE_GROUP> -n <PG_NAME>` (wait until `state=Ready`).
   - Evidence: v2/docs/worklog/2026-06-24.md lines 78-90 (BUG-0082 recovery); v2/docs/worklog/2026-06-19.md line 20 (`state=Stopped` → start → `Ready`); v2/docs/worklog/2026-06-20.md line 31 ("auto-stops every idle period — `az postgres flexible-server start -g <RESOURCE_GROUP> -n psql-<SUFFIX>` at session start").

2. **PRE-2 — Set the working directory to `v2/` (do NOT rely on `-C`/`--cwd`).** The repo has **two** azd manifests: the repo-root `azure.yaml` is **v1** (`name: chat-with-your-data-solution-accelerator`, azure.yaml line 3) and only `v2/azure.yaml` is **v2** (`name: chat-with-your-data-v2`, v2/azure.yaml line 22). azd resolves its project from the current directory, so an azd command launched from the repo root targets the **v1** template. Always `Set-Location <repo>/v2` first; the worklog and README both run azd from `v2/`.
   - Evidence: azure.yaml line 3 (v1 name); v2/azure.yaml line 22 (v2 name); v2/docs/worklog/2026-06-24.md line 123 ("redeployed with `azd deploy backend` (run from `v2/`)"); v2/README.md lines 47, 65 (`cd v2` before azd).
   - Note: async-launched VS Code terminals start at the **repo root**, not `v2` — a `Set-Location` prefix can be stripped by the command simplifier; verify the prompt cwd is `…\v2` before invoking azd (v2/docs/worklog/2026-06-12.md line 230).

3. **PRE-3 — Regenerate the function deploy artifact if deploying the function (BUG-0058 / BUG-0078).** `azd deploy function` has shipped a **stale** `v2/build-functions/` artifact because the `prepackage` hook did not fire on the per-service deploy path. Manually run the prepackage script first so current `src/` code (and all blueprints, incl. `blob_event`) is staged:
   - `uv run python scripts/prepackage_function.py` (from `v2/`), then confirm the staged subpackages line lists every blueprint (`add_url, batch_push, batch_start, blob_event, search_skill, core`).
   - Evidence: v2/docs/bugs.md line 117 (BUG-0058); v2/docs/bugs.md line 137 (BUG-0078 allow-list); v2/docs/worklog/2026-06-20.md line 50 (resume steps).

4. **PRE-4 — Open the function/documents storage account network before deploying the function or seeding sample data.** The function package/documents storage account is hardened `publicNetworkAccess=Disabled` (or `defaultAction=Deny`). `azd deploy function` uploads the package zip as a blob from the operator's workstation over the public internet → `403 InaccessibleStorageException / BlobUploadFailedException`. RBAC is not the gap. Open before, re-lock after (the runtime reaches storage via its private path, so re-locking does not break ingestion):
   - Open: `az storage account update -n st<SUFFIX> -g <RESOURCE_GROUP> --public-network-access Enabled --default-action Allow`
   - Re-lock after deploy + seed: `az storage account update -n st<SUFFIX> -g <RESOURCE_GROUP> --public-network-access Disabled`
   - Evidence: v2/docs/worklog/2026-06-25.md lines 62-67 (gotcha 2); v2/docs/bugs.md line ~ (BUG-0062, `--default-action Allow`).
   - **NOTE (postgresql mode):** in the postgresql deployment, Azure AI Search is NOT deployed; the function still uses the storage account for package + documents, so PRE-4 still applies whenever the function is redeployed.

5. **PRE-5 — Know which live overrides get reverted by `azd provision`/`azd deploy` and re-apply them.** Several durable Bicep back-ports are still pending; if you run full `azd provision` (not just per-service `azd deploy`), the bicep-derived config re-applies and reverts live patches. Re-apply (or land the bicep fix) after, or the backend/function re-break. Known live overrides:
   - ACR ARM-audience auth policy; backend Container App ACR `registries:` binding; Function App storage firewall (BUG-0062); backend postgres connect-username `AZURE_POSTGRES_ADMIN_PRINCIPAL_NAME=id-<SUFFIX>` (BUG-0063); orchestrator `CWYD_ORCHESTRATOR_NAME=langgraph` for pgvector (BUG-0064); KB connection name (BUG-0059, now seeded at post-provision per 2026-06-25); `AZURE_ENVIRONMENT=production` (BUG-0069, now bicep-wired).
   - Evidence: v2/docs/worklog/2026-06-17.md line 48 (re-apply-or-backport list); v2/docs/bugs.md BUG-0062/0063/0064/0069 rows.
   - **For pgvector specifically:** confirm the live backend env carries `CWYD_ORCHESTRATOR_NAME=langgraph` (pgvector's only coherent orchestrator) and `AZURE_POSTGRES_ADMIN_PRINCIPAL_NAME=id-<SUFFIX>` (the UAMI principal, NOT the human deployer UPN) — a full provision can revert these to the broken bicep defaults (BUG-0063, BUG-0064).

6. **PRE-6 — Decide whether a full provision is needed at all.** For a code-only redeploy, prefer per-service `azd deploy <service>` (backend / frontend / function) over `azd up` / `azd provision`, to avoid PRE-5 reverts. Provision only when infra/bicep changed.

---

## Per-Service Deploy Gotchas

### Backend (`azd deploy backend`)
- Run from `v2/`. Typical time ~2.5 min; produces a new revision at 100% traffic. (v2/docs/worklog/2026-06-24.md lines 123, 180).
- **Hard dependency on Postgres being `Ready`** (PRE-1). The FastAPI lifespan opens the asyncpg pool eagerly and **before `yield`**; if the DB is unreachable the new revision never finishes startup → crash-loop (BUG-0082). See the BUG-0082 special note below.
- The backend deploy is ACR image push (does NOT touch the function storage account), so PRE-4 does not apply to backend.
- After a full provision, re-verify the BUG-0063/0064 env vars (PRE-5).

### Frontend (`azd deploy frontend`) — BUG-0081 (open, high)
- **`azd deploy frontend` is effectively a no-op** — the App Service stays on the placeholder image even though azd reports SUCCESS. Root cause: `azure.yaml` declares the frontend as `host: appservice` + `docker:` block, but the azd `appservice` host **does not support `docker:`**, so azd silently zip-deploys code instead of building/pushing the image — a no-op against the container-kind App Service that bicep provisions.
- **Operator-endorsed manual container path (current workaround):** build `docker/Dockerfile.frontend` (`--target prod`, `--build-arg VITE_BACKEND_URL=<backend-fqdn>`) → push to ACR → `az webapp config container set` + enable UAMI ACR pull (`acrUseManagedIdentityCreds` / `acrUserManagedIdentityID=<UAMI-client-id>`) → restart. Result: `HTTP 200`, `<title>CWYD v2</title>`, placeholder gone.
- **The manual `az webapp config container set` is reverted by the next `azd provision`** — re-apply after any provision. Durable fix (move frontend to a Container App, or switch to azd code/static deploy) is a pending structural decision.
- Evidence: v2/docs/bugs.md line 140 (BUG-0081); v2/docs/worklog/2026-06-24.md lines ~95-110, 123-130.

### Function (`azd deploy function`) — BUG-0058, BUG-0080, BUG-0062, PRE-4
- **Run `scripts/prepackage_function.py` first** (PRE-3) or it ships stale code while reporting SUCCESS (BUG-0058) and may drop new blueprints (BUG-0078).
- **Open the storage account network first** (PRE-4) or the package upload 403s (BUG-0062).
- **Dependency-resolution / Python 3.11 host gotcha (BUG-0080, fixed):** `pyproject.toml` must pin `agent-framework-core==1.7.0` (NOT the umbrella `agent-framework`), because the umbrella pulls `agent-framework-hyperlight` → unresolvable `hyperlight-sandbox-backend-wasm` on the Functions host's Python 3.11 → Oryx remote build backtracks forever. Local dev (Python 3.14) never reproduces it. A guard test (`tests/functions/test_agent_framework_core_not_meta.py`) asserts the umbrella never returns. The repin also fixes the standard remote-build path for the next `azd up`.
- **Fallback deploy path when Oryx is slow:** build deps into `build-functions/.python_packages/lib/site-packages` in the `mcr.microsoft.com/azure-functions/python:4-python3.11` container, then `func azure functionapp publish func-<SUFFIX> --no-build --python` (logs `Skipping oryx build (remotebuild = false)`, ~4.5 min). (v2/docs/worklog/2026-06-24.md lines ~30-55; v2/docs/bugs.md line 139.)
- **Queue trigger needs an always-ready instance (BUG-0053, fixed via `alwaysReady`):** Flex Consumption per-function scaling won't wake the identity-based queue trigger from zero; an always-ready instance (`az functionapp scale config always-ready set`) is required for `batch_push` / `blob_event` to drain the queue.
- `azd deploy function` has exceeded azd's 20-min wait on Flex remote-build slowness; use `azd deploy function --timeout 2400` or the `--no-build` path. (v2/docs/worklog/2026-06-20.md lines 38, 50.)

---

## Post-Deploy Verification

### 1. Backend liveness — `GET /api/health` (the real signal)
- Endpoint: `GET https://<BACKEND_CA_FQDN>/api/health` — **always returns HTTP 200**; severity is in the body `status` field (`pass`/`fail`). (v2/src/backend/routers/health.py lines 5-50.)
- **Key nuance:** the health checks are **shallow** — `_check_database` only verifies the endpoint env var is *configured*, it does NOT round-trip the DB (v2/src/backend/services/health.py lines 21-31). So a `status: pass` body does NOT by itself prove DB connectivity. **The real liveness proof is that `/api/health` RESPONDS AT ALL**: the lifespan opens the asyncpg pool before `yield`, so if the endpoint answers, the lifespan completed → DB was reachable at startup. A hung/timing-out `/api/health` is the BUG-0082 crash-loop signature.
- Readiness probe: `GET /api/health/ready` returns **HTTP 503** when any required check FAILs (used by ACA to pull the pod). A healthy deploy returns 200. (v2/src/backend/routers/health.py lines 44-50.)
- Healthy body shape: `status: pass`, `auth_enforced: true` (when `environment == production`), and `checks` = `[foundry_iq: pass, database: pass (db_type=postgresql), search: skip (index_store=pgvector — no separate search service)]`. Note **`search: skip` is correct in pgvector mode** and does NOT drag the overall status down. (v2/src/backend/services/health.py lines 33-66.)

### 2. Configuration snapshot — `GET /api/admin/status`
- Endpoint: `GET https://<BACKEND_CA_FQDN>/api/admin/status` (admin-gated; v2/src/backend/routers/admin.py lines 115-150).
- A healthy pgvector deploy returns (fields from `AdminStatus`, v2/src/backend/models/admin.py lines 100-122):
  - `orchestrator_name`: `langgraph` (the **effective** value — env/code default overlaid with any persisted `RuntimeConfig` override via `resolve_effective_config`; BUG-0068 fixed this to match what the deployment runs). For pgvector this must be `langgraph` (BUG-0064); `agent_framework` also works on pgvector via app-side retrieval (BUG-0066) but `langgraph` is the bicep default.
  - `db_type`: `postgresql`
  - `index_store`: `pgvector`
  - `environment`: `production` (NOT `local` — if it shows `local`, the `AZURE_ENVIRONMENT` env var is unwired and the admin local-dev auth bypass could fail-open; BUG-0069).
  - `foundry_project_endpoint_host`: the Foundry project host (host only, no path)
  - `gpt_deployment`, `embedding_deployment`, `reasoning_deployment`: the configured deployment names
  - `search_enabled`: `false` in pgvector mode (no Azure AI Search endpoint)
  - `app_insights_enabled`: `true` when the App Insights connection string is set (note BUG-0055 — telemetry historically did not arrive even when this is `true`)
  - `cors_origins`: list of allowed origins
  - `version`: app version
- Sensitive fields (UAMI ids, tenant id, full connection strings, OpenAI API version) are deliberately omitted (locked by `test_status_does_not_leak_sensitive_settings`). (v2/src/backend/models/admin.py lines 101-108.)

### 3. Functional probes (after function + frontend redeploy)
- Admin documents list: `GET /api/admin/documents` → 200 with the existing sources (a 500 here in pgvector mode usually means Postgres is stopped — PRE-1). (v2/docs/worklog/2026-06-20.md line 31.)
- Chat round-trip grounds + cites (pgvector dense retrieval via `langgraph`, BUG-0065 fixed; or `agent_framework` app-side, BUG-0066).
- Frontend: `HTTP 200` + `<title>CWYD v2</title>` (NOT the App Service placeholder "Hey, App Service developers!") — confirms the BUG-0081 manual container deploy is live.
- Function: `az functionapp function list` shows all 6 functions (`add_url, batch_push, batch_start, blob_event, health, search_skill`); function `/api/health` → `{"status":"ok"}`. (v2/docs/bugs.md line 139.)

---

## BUG-0082 — Special note (Postgres stopped → backend crash-loop)

**This is the single highest-risk redeploy hazard for the PostgreSQL environment.**

- **Symptom:** backend Container App reports `Running` but `/api/health` times out; the replica has `Started: False` with the default startup probe failing 200+ times; console logs stop at uvicorn's `Waiting for application startup.` (the app's own log lines never flush because stdout is block-buffered in the container).
- **Root cause:** the FastAPI lifespan's first DB calls — `get_runtime_config` → `ensure_pool` → `ensure_schema` — have **NO connect timeout**. Confirmed in code: `PostgresClient._ensure_pool` calls `asyncpg.create_pool(dsn=..., user=..., password=..., min_size=1, max_size=10)` with **no `timeout=` argument** (v2/src/backend/core/providers/databases/postgres.py lines 298-360, specifically the `create_pool` call at lines 322-328). An unreachable server (auto-stopped Burstable instance, firewall drop) hangs the lifespan **forever** — the app never finishes startup, the probe restarts it indefinitely instead of failing fast or degrading.
  - The lifespan calls this path: `app.state.runtime_overrides = await database_client.get_runtime_config()` (v2/src/backend/app.py line 128) and, in pgvector mode, `await database_client.ensure_pool()` + `await search_provider.ensure_schema()` (v2/src/backend/app.py lines 164-182) — all before `yield` (app.py line 192). Any of these hangs forever if Postgres is unreachable.
- **Recovery (observed 2026-06-24):**
  1. `az postgres flexible-server start -g <RESOURCE_GROUP> -n <PG_NAME>` (wait for `state=Ready`).
  2. Restart the backend revision: `az containerapp revision restart -g <RESOURCE_GROUP> -n <BACKEND_CA_NAME> --revision <REVISION>` (or `az containerapp revision list` to find the active revision; a restart re-runs the lifespan, which now connects).
  3. Verify `/api/health` → `status: pass`.
- **Status:** open (root-caused + recovery documented 2026-06-24; durable code fix pending). Fix direction (not yet implemented): add a bounded connect timeout to the asyncpg connect / pool acquisition and let the lifespan fail fast (or mark the `database` check degraded) so a transient DB outage can't become a permanent crash-loop; add `PYTHONUNBUFFERED=1` to the backend image so startup logs flush.
- **Redeploy implication:** **Always do PRE-1 (start Postgres, confirm `Ready`) BEFORE `azd deploy backend`.** Deploying the backend while Postgres is stopped guarantees a crash-loop, and the new revision will not self-heal until both the DB is started AND the revision is restarted.
- Evidence: v2/docs/bugs.md line 141 (BUG-0082 row); v2/docs/worklog/2026-06-24.md lines 78-90; v2/src/backend/core/providers/databases/postgres.py lines 298-360; v2/src/backend/app.py lines 66-192.

---

## Other bug details (pre-checks they imply)

| Bug | Status | Symptom | Root cause | Redeploy pre-check implied |
| --- | --- | --- | --- | --- |
| BUG-0082 | open (medium) | Backend crash-loops; `/api/health` times out; replica `Started: False` | Lifespan DB connect (`get_runtime_config`→`ensure_pool`→`ensure_schema`) has no connect timeout; auto-stopped Burstable Postgres hangs startup forever | **PRE-1: start Postgres + confirm `Ready` before backend deploy.** Recovery = `flexible-server start` + `containerapp revision restart`. |
| BUG-0081 | open (high) | Frontend stays on App Service placeholder; `azd deploy frontend` reports SUCCESS but is a no-op | azd `appservice` host doesn't support `docker:`; azd zip-deploys code against a container-kind App Service | Deploy frontend via the **manual container path** (build `Dockerfile.frontend`, push ACR, `az webapp config container set`, restart). Re-apply after any `azd provision`. |
| BUG-0058 | open (medium) | `azd deploy function` ships stale `build-functions/` while reporting SUCCESS | `prepackage` hook does not run on the per-service deploy path | **PRE-3: run `uv run python scripts/prepackage_function.py` before `azd deploy function`.** |
| BUG-0055 | open (medium) | App Insights `appi-<SUFFIX>` has received **zero** telemetry ever (function host + backend) | OpenTelemetry / App Insights export unwired or misconfigured at both runtimes despite `APPLICATIONINSIGHTS_CONNECTION_STRING` present | Don't rely on App Insights for post-deploy diagnosis; use `FunctionAppLogs` in the `log-<SUFFIX>` Log Analytics workspace (populated via `allLogs` diagnostic) + the backend `/api/health` + container console logs instead. `app_insights_enabled:true` in `/api/admin/status` does NOT guarantee telemetry flows. |
| BUG-0054 | open (medium) | `doc-processing-poison` holds base64 Event Grid `BlobCreated` envelopes (schema mismatch poison) | A stray/legacy Event Grid subscription delivered raw `BlobCreated` events into `doc-processing`; `batch_push` expects CWYD ingestion envelopes. Fix = `blob_event` translator + repoint EG to `blob-events` queue + `ingestion_trigger` flag. | Noise/poison only — does NOT block ingestion (backend stays on `direct_enqueue`). Resume steps: drain the 10 poison messages, deploy function (after PRE-3), then flip `AZURE_ENV_INGESTION_TRIGGER` → `event_grid`. `blob_event` queue trigger needs the BUG-0053 always-ready instance. |

- Evidence rows: v2/docs/bugs.md lines 113 (BUG-0054), 114 (BUG-0055), 117 (BUG-0058), 140 (BUG-0081), 141 (BUG-0082).

---

## Key file references

- Two azd manifests (cwd gotcha): azure.yaml line 3 (v1 `name`); v2/azure.yaml line 22 (v2 `name`).
- Backend lifespan + DB connect path: v2/src/backend/app.py lines 66-192 (esp. line 128 `get_runtime_config`, lines 164-182 `ensure_pool` / `ensure_schema`, line 192 `yield`).
- Postgres pool creation, **no connect timeout**: v2/src/backend/core/providers/databases/postgres.py lines 298-360 (`_ensure_pool`; `asyncpg.create_pool` call lines 322-328 — args `dsn`, `user`, `password`, `min_size`, `max_size`; no `timeout`).
- `ensure_pool` public wrapper: v2/src/backend/core/providers/databases/postgres.py lines 278-286.
- Health router (always-200 diagnostic + 503 readiness): v2/src/backend/routers/health.py lines 1-50.
- Health checks (shallow, config-only): v2/src/backend/services/health.py lines 13-66.
- Admin status endpoint (effective orchestrator): v2/src/backend/routers/admin.py lines 115-150.
- `AdminStatus` model (returned fields): v2/src/backend/models/admin.py lines 100-122.
- Worklogs: v2/docs/worklog/2026-06-24.md (BUG-0080/0081/0082/0083/0084, backend+frontend redeploy); v2/docs/worklog/2026-06-25.md (KB-MCP seeder, deploy gotchas 1-2); v2/docs/worklog/2026-06-20.md line 31 (Postgres auto-stop); v2/docs/worklog/2026-06-19.md line 20 (`state=Stopped` recovery); v2/docs/worklog/2026-06-17.md line 48 (live-override re-apply list).
- Defect registry: v2/docs/bugs.md.

---

## Status

**Complete** — all five research questions answered from repo files. No live Azure calls were made; all identifiers are placeholders.

## Recommended next research (not done this session)

- [ ] Confirm the exact `az containerapp revision restart` invocation syntax the operator used (revision name vs `--revision` lookup) — worklog says "a backend revision restart" without the literal command; verify against az CLI docs.
- [ ] Confirm the Burstable auto-stop window precisely (worklog says "~7 idle days" in one place and "every idle period" in another) — check the Flex Server SKU config / Azure docs.
- [ ] Verify whether `AZURE_ENV_INGESTION_TRIGGER` is currently `direct_enqueue` or `event_grid` on the live backend (affects whether the BUG-0054 EG path is active) — requires a live `az containerapp` env read.
- [ ] Check whether the BUG-0082 code fix (bounded connect timeout + `PYTHONUNBUFFERED=1`) has landed since 2026-06-24 — grep `postgres.py` for a later `timeout=` addition and the backend Dockerfile for `PYTHONUNBUFFERED`.
- [ ] Determine the durable BUG-0081 decision (Container App vs static deploy) before writing the frontend section of the runbook as permanent.

## Clarifying questions

1. **Scope of the runbook redeploy:** code-only per-service `azd deploy` (recommended, avoids PRE-5 reverts) or full `azd up`/`azd provision`? The pre-check list differs (a full provision reverts the live overrides in PRE-5).
2. **Which services** does this redeploy cover — backend only, or backend + frontend + function? Frontend (BUG-0081) and function (PRE-3/PRE-4) carry their own manual steps.
3. **Is the BUG-0082 connect-timeout code fix expected to be in place** for this runbook, or should the runbook assume the current no-timeout behavior (and therefore hard-gate on PRE-1)?
