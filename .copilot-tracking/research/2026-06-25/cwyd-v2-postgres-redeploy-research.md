<!-- markdownlint-disable-file -->
# Task Research: CWYD v2 Redeploy with PostgreSQL on Existing v2/.azure Resources

Research what is required to redeploy the latest CWYD v2 changes to the existing Azure
environment (azd env under `v2/.azure`) running the PostgreSQL / pgvector configuration.

## Task Implementation Requests

* Read the latest CWYD v2 changes (working tree + recent commits) and determine which
  deployable units (backend, frontend, function) are affected and therefore need redeploy.
* Determine the exact redeploy procedure against the existing `v2/.azure` azd environment
  configured for PostgreSQL (db_type=postgresql, index_store=pgvector).
* Surface any known redeploy gotchas / pre-checks (DB stopped, azd cwd behavior, env vars).

## Scope and Success Criteria

* Scope: redeploy of the already-provisioned PostgreSQL azd env; covers both the code-only
  `azd deploy <service>` path and the full `azd provision` / `azd up` path. NOT a fresh
  greenfield environment, NOT cross-cloud migration.
* Assumptions:
  * The single `v2/.azure/<AZD_ENV_NAME>` azd environment exists and is configured for
    PostgreSQL — confirmed by `.env`: `AZURE_DB_TYPE=postgresql`, `AZURE_INDEX_STORE=pgvector`,
    Cosmos/Search keys empty.
  * PostgreSQL Flexible Server `psql-<SUFFIX>` (Burstable Standard_B2s, v16, db `cwyd`,
    Entra-only auth) with pgvector is the active backend store.
* Success Criteria:
  * Ordered redeploy runbook: pre-checks → per-service deploy → verify.
  * Explicit per-unit "needs redeploy" determination with justification.
  * Documented gotchas (azd cwd, Postgres auto-stop / BUG-0082, frontend no-op / BUG-0081,
    function prepackage + storage firewall, live-override reverts).

## Outline

* Current state: working tree clean, HEAD pushed; "latest changes" live in commits since 06-23.
* PostgreSQL env was last deployed ~06-17 → predates every recent change → all 3 units + infra stale.
* azd env + service mapping + PostgreSQL-selecting parameters.
* Two redeploy strategies (per-service deploy vs full provision) + recommendation.
* Pre-checks, per-service gotchas, post-deploy verification, BUG-0082 special note.

## Potential Next Research

* Verify the PostgreSQL env's ACTUAL live state (image tags, current revision, current
  `CWYD_ORCHESTRATOR_NAME` / `AZURE_POSTGRES_ADMIN_PRINCIPAL_NAME`, Postgres `Ready`/`Stopped`)
  via `az`/`azd` rather than inferring from worklogs.
  * Reasoning: confirms which live overrides are still in place before a full provision reverts them.
  * Reference: subagent files (all three) flag this as the top live-verification gap.
* Confirm whether the BUG-0082 durable fix (bounded asyncpg connect timeout + `PYTHONUNBUFFERED=1`)
  has landed; if not, PRE-1 (start Postgres) is a hard gate before `azd deploy backend`.
  * Reference: v2/src/backend/core/providers/databases/postgres.py:322-328 (no `timeout=`).
* Regression-check the postgresql `azd up` path against the 06-25 infra changes (KB-MCP module
  removal → post_provision seeder, new hooks) — BUG-0060 / BUG-0061 history.

## Research Executed

### File Analysis

* v2/azure.yaml:104-157 — three deployable services: `backend` (containerapp, `ca-backend-<SUFFIX>`,
  ACR remote build), `frontend` (appservice, `app-frontend-<SUFFIX>`, build-from-source + prepackage
  hook), `function` (function, `func-<SUFFIX>`, project path `./build-functions`, prepackage hook).
* v2/azure.yaml:206-228 — project hooks: `postprovision` → post-provision.{sh,ps1} (pgvector extension
  + KB-MCP seeder); `postdeploy` → upload-sample-data.{sh,ps1}.
* v2/infra/main.bicep:86 — `param databaseType` (`cosmosdb` | `postgresql`); the single mode switch.
* v2/infra/main.bicep:1488-1540 — PostgreSQL AVM flexible-server 0.15.3 deployed `if (databaseType ==
  'postgresql')`; pgvector via `azure.extensions = VECTOR` (1534-1540).
* v2/infra/main.bicep:1623 — `var indexStoreValue = databaseType == 'cosmosdb' ? 'AzureSearch' : 'pgvector'`.
* v2/infra/main.bicep:1884 — `CWYD_ORCHESTRATOR_NAME = databaseType == 'postgresql' ? 'langgraph' :
  'agent_framework'` — **postgresql default is langgraph, NOT agent_framework**.
* v2/infra/main.parameters.json:18-20 — `databaseType` ← `${AZURE_ENV_DATABASE_TYPE=cosmosdb}` (env = postgresql).
* v2/src/backend/app.py:66-192 — FastAPI lifespan; DB connect path `get_runtime_config` (128) →
  `ensure_pool` / `ensure_schema` (164-182) all BEFORE `yield` (192).
* v2/src/backend/core/providers/databases/postgres.py:298-360 — `_ensure_pool`; `asyncpg.create_pool`
  call (322-328) has **no `timeout=`** → BUG-0082 root cause.
* v2/src/backend/routers/health.py:1-50 — `/api/health` always-200 + `/api/health/ready` 503-on-fail.
* v2/src/backend/services/health.py:13-66 — health checks are shallow (config-only, no DB round-trip).
* v2/src/backend/models/admin.py:100-122 — `AdminStatus` returned fields.

### Code Search Results

* `git status --short` → working tree clean except one untracked tracking-doc. No uncommitted source/infra.
* `git rev-parse --abbrev-ref HEAD` → `feature/cwyd-v2-6626` (pushed, = origin).
* `git log --oneline` recent window (06-23 → 06-25): `63907b7`, `638594e`, `dc112dd`, `2428a0b`,
  `558c3c9`, `afb6071`, `46bdc9b`.
* `grep "azd deploy"` across v2/** → commands documented only in azure.yaml comments, worklogs, bugs.md
  (NOT in Makefile/README).

### External Research

* None required — repo-internal redeploy of existing resources.

### Project Conventions

* Standards referenced: `.github/copilot-instructions.md` Hard Rules (#18 no env-specific values in
  tracked files; azd-only, no ARM "Deploy to Azure" button).
* Instructions followed: Task Researcher mode (research-only, subagent-delegated). All real Azure IDs
  redacted to placeholders in this and all subagent files.

## Key Discoveries

### Project Structure

* **Single azd env** under v2/.azure (`<AZD_ENV_NAME>`, also the `defaultEnvironment`); per-env
  `config.json` is empty `{}`; the `.env` snapshot is what `azd deploy` reads.
* **PostgreSQL mode confirmed** in `.env`: `AZURE_DB_TYPE=postgresql`, `AZURE_ENV_DATABASE_TYPE=postgresql`,
  `AZURE_INDEX_STORE=pgvector`; `AZURE_AI_SEARCH_*` / `AZURE_COSMOS_*` empty (those services not deployed).
* **Two competing azure.yaml manifests** — repo-root `azure.yaml` is **v1**
  (`chat-with-your-data-solution-accelerator`); only `v2/azure.yaml` is **v2**. azd resolves its
  project from cwd → every v2 command MUST run from `v2/`.

### Implementation Patterns — which units need redeploy

The working tree is clean and HEAD is pushed, so "redeploy" = "bring the PostgreSQL cloud env up to HEAD".
The PostgreSQL env was last provisioned/deployed **~2026-06-17**, which **predates every recent change**.

| Unit | Needs redeploy? | Why (functional changes since ~06-17) |
|---|---|---|
| **Backend** (Container App) | **YES** | Assistant-type presets / ADR 0030 (46bdc9b); BUG-0084 RAI calibration (`PROMPT_REVIEW_AGENT` + `validate_prompt_with_rai` allow-list); BUG-0076/0083 fixes. (06-25 docstring scrub is cosmetic but rides along.) |
| **Frontend** (App Service) | **YES** (manual container path — BUG-0081) | App Service SPA host (frontend_app.py); runtime-config bootstrap (App.tsx + new api/runtimeConfig.tsx); `backendUrl` + user-identity header wiring across api/*; assistant-type dropdown + BUG-0083 normalizer. |
| **Function** (Flex Consumption) | **YES** (prepackage + open storage first) | host.json `messageEncoding=none` (dc112dd, BUG-0056); App Insights telemetry wiring (afb6071, BUG-0055); critical `agent-framework-core` repin (558c3c9, BUG-0080) that unblocks the build. |
| **Infra** (Bicep / hooks) | **YES — needs `azd provision`** | main.bicep / main.parameters.json / azure.yaml changed; KB-MCP Bicep module removed → post_provision seeder; new prepackage/postprovision/postdeploy hooks. A bare per-service deploy will NOT apply these. |

### Orchestrator note (important)

* Bicep default for postgresql mode is **`langgraph`** (main.bicep:1884), not `agent_framework`.
  Both orchestrators register and are runtime-switchable; agent_framework also works on pgvector
  app-side (ADR 0027 / BUG-0066). If the live backend currently shows `agent_framework`, that is a
  live `--set-env-vars` override and **a full `azd provision` would revert it to `langgraph`** — verify
  before provisioning.

### Open bugs that gate / caveat this redeploy (6)

| ID | Area | Sev | Redeploy impact |
|---|---|---|---|
| BUG-0082 | backend | medium | **Start Postgres + confirm `Ready` BEFORE `azd deploy backend`** — stopped Burstable DB + no lifespan connect timeout = permanent crash-loop. |
| BUG-0081 | infra | high | `azd deploy frontend` is a **no-op**; use manual container path (build Dockerfile.frontend → push ACR → `az webapp config container set` → restart). Reverted by next `azd provision`. |
| BUG-0058 | functions | medium | Run `uv run python scripts/prepackage_function.py` before `azd deploy function` (per-service deploy may skip the prepackage hook → stale artifact). |
| BUG-0062 | functions | — | Temporarily open `st<SUFFIX>` public network before `azd deploy function` (package upload 403 against private-only storage); re-lock after. |
| BUG-0055 | infra | medium | App Insights receives zero telemetry — don't rely on it for post-deploy diagnosis; use Log Analytics + `/api/health` + container console. |
| BUG-0054 | infra | medium | Event Grid poison messages — noise only, does NOT block ingestion (stays on `direct_enqueue`). |

### Complete Examples — exact redeploy commands (run from v2/)

```powershell
# ── PRE-CHECK: Postgres must be Ready before backend (BUG-0082) ──
az postgres flexible-server show  -g <RESOURCE_GROUP> -n <PG_NAME> --query state -o tsv   # expect: Ready
az postgres flexible-server start -g <RESOURCE_GROUP> -n <PG_NAME>                         # if Stopped

# ── ALWAYS cd into v2 first (repo root is the v1 azure.yaml) ──
Set-Location c:\workstation\Microsoft\github\cwyd-pg\v2

# Backend (Container App) — ACR remote build + revision swap, ~2.5 min
azd deploy backend  -e <AZD_ENV_NAME> --no-prompt

# Function (Flex) — regenerate artifact + open storage first, slow remote build
uv run python scripts/prepackage_function.py
az storage account update -n st<SUFFIX> -g <RESOURCE_GROUP> --public-network-access Enabled --default-action Allow
azd deploy function -e <AZD_ENV_NAME> --no-prompt --timeout 2400
az storage account update -n st<SUFFIX> -g <RESOURCE_GROUP> --public-network-access Disabled

# Frontend — azd deploy is a NO-OP (BUG-0081); use the manual container path instead:
#   docker build -f docker/Dockerfile.frontend --target prod --build-arg VITE_BACKEND_URL=<backend-fqdn> -t cr<SUFFIX>.azurecr.io/frontend:<tag> .
#   docker push cr<SUFFIX>.azurecr.io/frontend:<tag>
#   az webapp config container set ... (UAMI ACR pull) ; az webapp restart ...
```

### Configuration Examples — healthy post-deploy `/api/admin/status` (pgvector)

```json
{
  "orchestrator_name": "langgraph",
  "db_type": "postgresql",
  "index_store": "pgvector",
  "environment": "production",
  "search_enabled": false,
  "gpt_deployment": "gpt-5.1",
  "embedding_deployment": "text-embedding-3-large",
  "reasoning_deployment": "o4-mini",
  "app_insights_enabled": true,
  "version": "2.0.0"
}
```

## Technical Scenarios

### Scenario A — Full `azd provision` + `azd deploy --all` (i.e., `azd up`) — RECOMMENDED

The PostgreSQL env is stale across **infra + all three services** (last deployed ~06-17). Since then
main.bicep, main.parameters.json, azure.yaml hooks, and the KB-MCP→seeder change all landed. A bare
per-service `azd deploy` would update the running code but **silently skip every infra/hook change**.

**Requirements:**

* Postgres `Ready` (PRE-1) before the backend container boots.
* Run from `v2/` (PRE-2).
* Re-apply live overrides that a fresh provision reverts (PRE-5): at minimum
  `AZURE_POSTGRES_ADMIN_PRINCIPAL_NAME=id-<SUFFIX>` (BUG-0063), the backend ACR `registries:` binding /
  ARM-audience auth, the function storage firewall (BUG-0062), and the BUG-0081 frontend container.
  Note `CWYD_ORCHESTRATOR_NAME=langgraph` is now Bicep-wired for postgresql (main.bicep:1884) so it is
  NO longer a manual re-apply.
* Function prepackage (PRE-3) + open storage (PRE-4); frontend manual container (BUG-0081).

**Preferred Approach:**

* Run `azd provision` then `azd deploy --all` from `v2/` (or `azd up`), in this order:
  1. PRE-1 start Postgres. PRE-2 `Set-Location v2`.
  2. `azd provision -e <AZD_ENV_NAME> --no-prompt` (applies Bicep/param/hook changes; runs postprovision
     pgvector + KB-MCP seeder).
  3. PRE-3 `prepackage_function.py`; PRE-4 open `st<SUFFIX>`.
  4. `azd deploy backend` → `azd deploy function` → re-lock storage.
  5. Frontend via manual container path (BUG-0081).
  6. Re-apply PRE-5 live overrides; verify.

```text
v2/
  azd provision        # Bicep + params + postprovision hook (pgvector ext, KB-MCP seeder)
  azd deploy backend   # Container App
  azd deploy function  # Flex (prepackage + open-storage first)
  (manual) frontend    # Dockerfile.frontend -> ACR -> az webapp config container set
```

**Implementation Details:**

* Rationale: infra drift since 06-17 is real and a per-service deploy cannot close it. The KB-MCP
  module removal + new hooks must run through `azd provision` to converge the env.
* Cost: provision reverts live overrides (PRE-5) → must re-apply; longer wall-clock; small RBAC
  back-port collision risk (06-25 Gotcha 1 — delete the stray manual role assignment, re-run).

#### Considered Alternatives

Scenario B (below) is faster but does not apply the infra/hook changes.

### Scenario B — Per-service `azd deploy` only (code-only, no provision)

Deploy just the changed service code (`azd deploy backend` / `function`, manual frontend) and skip
`azd provision` entirely.

**Requirements:** same PRE-1/2/3/4 + BUG-0081 manual frontend, but **no PRE-5 re-apply** (nothing is
reverted because Bicep is never re-run).

**Preferred Approach:** use this ONLY if the operator confirms the infra/Bicep/hook changes are NOT
needed in the PostgreSQL env (e.g., they only want the BUG-0084 RAI fix + assistant presets live).

**Why not the default:** the env is stale on infra too (removed KB-MCP module, new hooks, new params).
Skipping provision leaves those un-applied — acceptable for a quick code-only refresh, but it does NOT
bring the env fully to HEAD. Best framed as a fast path, not the complete redeploy.

**Trade-offs:** ✅ fast, no live-override reverts, lowest blast radius. ❌ misses all infra/hook
changes; the env stays partially behind HEAD.

## Open Questions for the User (materially change the runbook)

1. **Full bring-to-HEAD (Scenario A: `azd provision` + deploy all) or fast code-only refresh
   (Scenario B: `azd deploy` backend/function/frontend, no provision)?** Infra changed since 06-17, so
   only Scenario A fully converges the env — but it reverts live overrides that must be re-applied.
2. **Which services are in scope** — all three (backend + frontend + function), or backend only? Frontend
   (BUG-0081) and function (PRE-3/PRE-4) each carry manual steps.
3. **Orchestrator default:** keep `langgraph` (the Bicep default for postgresql) or pin
   `agent_framework`? If the live backend currently runs `agent_framework`, a full provision (Scenario A)
   will revert it to `langgraph` unless re-applied.
