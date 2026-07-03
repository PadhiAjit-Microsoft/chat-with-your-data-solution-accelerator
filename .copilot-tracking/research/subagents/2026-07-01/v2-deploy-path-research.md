<!-- markdownlint-disable-file -->
# Research: CWYD v2 First azd up — Deploy Path + Grounding

Status: Complete
Date: 2026-07-01

Scope: READ-ONLY research to arm a planner for the first `azd up` of CWYD v2.
No code modified; no deploy/provision commands run. Only local read-only
commands executed (`azd version`, file reads).

Source of truth = the actual files under `v2/` on disk as of 2026-07-01, NOT
prior tracking notes.

---

## 0. Executive summary / net answers

- **Sample-data grounding — RESOLVED.** A **fresh `azd up` in cosmosdb mode
  GROUNDS out-of-the-box.** There IS a project-level `postdeploy` hook in
  `v2/azure.yaml` that runs `upload_sample_data.py`, which uploads sample PDFs
  from the **repo-root `data/` folder** into the `documents` blob container and
  enqueues ingestion. The earlier infra survey
  (`.copilot-tracking/research/subagents/2026-06-25/v2-infra-current-state.md`)
  is **STALE** on this point. WI-03's note ("the `postdeploy` upload exists") is
  the correct one. Detail in §1.
- **`post_provision.py` still does NOT upload documents** — the 2026-06-25
  survey was right about *that script*. It only ensures the Search index schema
  + Foundry IQ KB. The document upload is a **separate** `postdeploy` hook. Both
  statements are compatible; the survey's error was concluding "no upload step
  anywhere in the v2 azd flow."
- **`azd version` = `1.27.0`** — satisfies the pin `>= 1.18.0 != 1.23.9`.
- **Deploy scope is `resourceGroup`, not subscription** (`targetScope =
  'resourceGroup'`, `v2/infra/main.bicep` line 25). The what-if pre-check is
  `azd provision --preview` (recommended) or a resource-group-scoped
  `az deployment group what-if` — NOT `az deployment sub what-if`. Detail in §5.
- **Silent-except gate is a RED test but NOT a deploy blocker.** `azd up` never
  runs pytest, so `test_no_silent_excepts` failing on
  `v2/src/functions/core/search_resolution.py` does not stop a deploy. It blocks
  a green pytest/CI lane. Detail in §10.

Ordered deploy blockers are in §11.

---

## 1. GROUND TRUTH — sample-data upload / grounding (Q1–Q3)

### 1.1 There IS a `postdeploy` hook (Q1)

`v2/azure.yaml` hooks block — quoted verbatim (project scope; there are NO
per-service hooks — the `services:` block has none):

```yaml
# v2/azure.yaml lines ~163-200
hooks:
  postprovision:
    posix:
      shell: sh
      run: ./scripts/post-provision.sh
      continueOnError: false
      interactive: true
    windows:
      shell: pwsh
      run: ./scripts/post-provision.ps1
      continueOnError: false
      interactive: true
  postdeploy:
    posix:
      shell: sh
      run: ./scripts/upload-sample-data.sh
      continueOnError: true
      interactive: true
    windows:
      shell: pwsh
      run: ./scripts/upload-sample-data.ps1
      continueOnError: true
      interactive: true
```

Hook enumeration (both scopes):

| Scope | Hook | Script (posix / windows) | continueOnError |
|---|---|---|---|
| project | `postprovision` | `scripts/post-provision.sh` / `scripts/post-provision.ps1` → `post_provision.py` | `false` |
| project | `postdeploy` | `scripts/upload-sample-data.sh` / `scripts/upload-sample-data.ps1` → `upload_sample_data.py` | `true` |
| per-service | (none) | `services.backend/frontend/function` declare **no** hooks | — |

There is **no** `preprovision`, `predeploy`, or `prepackage` hook anywhere in
`v2/azure.yaml`. The azure.yaml docstring header even documents that the
functions Dockerfile "reproduces the Functions deploy layout … so no prepackage
staging hook is needed."

The windows wrapper (`v2/scripts/upload-sample-data.ps1`) simply shells to the
Python script:

```powershell
# v2/scripts/upload-sample-data.ps1 (tail)
$scriptDir = Split-Path -Parent $MyInvocation.MyCommand.Path
& uv run python (Join-Path $scriptDir 'upload_sample_data.py') @args
exit $LASTEXITCODE
```

### 1.2 `upload_sample_data.py` DOES upload into the `documents` container (Q2)

`v2/scripts/upload_sample_data.py` — the module docstring states its purpose:

> "Post-deploy seed: upload sample documents and enqueue ingestion. … Runs
> after a successful `azd deploy` / `azd up` so chat grounds out-of-the-box
> without an operator manually uploading documents."

Blob-upload call site (`v2/scripts/upload_sample_data.py`, `upload_blob_if_absent`):

```python
def upload_blob_if_absent(container_client: ContainerClient, file_path: Path) -> bool:
    blob_client = container_client.get_blob_client(file_path.name)
    if blob_client.exists():
        return False
    blob_client.upload_blob(file_path.read_bytes(), overwrite=False)
    return True
```

It then enqueues an ingestion message (direct-enqueue mode) using the shared
`BatchPushQueueMessage` contract:

```python
def enqueue_ingest_message(queue_client: QueueClient, container_name: str, filename: str) -> None:
    message = BatchPushQueueMessage(container_name=container_name, filename=filename)
    queue_client.send_message(message.model_dump_json())
```

Target container / queue come from azd outputs:

```python
_ENV_ACCOUNT   = "AZURE_STORAGE_ACCOUNT_NAME"
_ENV_CONTAINER = "AZURE_DOCUMENTS_CONTAINER"   # the `documents` container
_ENV_QUEUE     = "AZURE_DOC_PROCESSING_QUEUE"
```

Behavior nuances that matter for "does a fresh deploy ground":

- **Non-interactive shell, no override → seeds the DEFAULT benefits PDF set.**
  `resolve_selection(...)` returns `AssistantType.DEFAULT` when `stdin` is not a
  TTY and neither `--set` nor `AZURE_ENV_SAMPLE_DATA` is set. So even fully
  unattended, chat grounds. Opt out with `AZURE_ENV_SAMPLE_DATA=none`.
- **Interactive shell → menu** (1 default / 2 contract / 3 employee / 4 all /
  0 skip; default key `4` = all).
- **Idempotent** — `upload_blob_if_absent` skips blobs already present; safe to
  re-run.
- **continueOnError: true** — a seed hiccup (e.g. RBAC propagation) never fails
  an otherwise-successful `azd up`.
- **Direct-enqueue only enqueues; event-grid mode suppresses enqueue** (blob
  write alone triggers ingestion) — `enqueue = trigger == DIRECT_ENQUEUE`.
- **Bounded post-seed poll** — when `AZURE_AI_SEARCH_ENDPOINT` +
  `AZURE_AI_SEARCH_INDEX` are set (cosmosdb mode), it polls
  `SearchClient.get_document_count()` up to `_INDEX_WAIT_TIMEOUT_S = 300.0` s and
  prints a PASS/FAIL banner.

The DEFAULT / EMPLOYEE benefits set (uploaded by basename):

```python
_BENEFITS_SET = (
    "Benefit_Options.pdf",
    "employee_handbook.pdf",
    "PerksPlus.pdf",
    "role_library.pdf",
    "Northwind_Standard_Benefits_Details.pdf",
    "Northwind_Health_Plus_Benefits_Details.pdf",
)
```

### 1.3 Where sample data lives + no `v2/data/` folder (Q3)

- **`v2/data/` does NOT exist** — `file_search` for `v2/data/**` returned no
  files. The 2026-06-25 survey was correct on that narrow fact.
- **Sample docs live in the repo-root `data/` folder.** The resolver:

  ```python
  def _curated_data_dir() -> Path:
      """Return the repo-root data/ folder that holds the sample corpus."""
      return Path(__file__).resolve().parents[2] / "data"
  ```

  `v2/scripts/upload_sample_data.py` → `parents[0]=v2/scripts`,
  `parents[1]=v2`, `parents[2]=<repo root>` → `<repo-root>/data`.

- Repo-root `data/` contents (confirmed via `list_dir`): `Benefit_Options.pdf`,
  `employee_handbook.pdf`, `PerksPlus.pdf`, `role_library.pdf`,
  `Northwind_Standard_Benefits_Details.pdf`,
  `Northwind_Health_Plus_Benefits_Details.pdf`, plus `MSFT_FY23Q4_10K.docx`,
  `PressReleaseFY23Q4.docx`, several `Woodgrove …` PDFs, and a
  `contract_data/` subdir (globbed by the `contract` / `all` scopes via
  `_CONTRACT_DIR = "contract_data"`). All six DEFAULT-set files are present, so
  the default seed resolves fully.

### 1.4 `post_provision.py` scope (NOT documents)

`v2/scripts/post_provision.py` responsibilities (docstring + code):

1. postgresql mode → `CREATE EXTENSION IF NOT EXISTS vector`.
2. cosmosdb mode → `_ensure_search_index(...)` creates the `cwyd-index` schema
   (id/content/title/url/content_vector + HNSW vector + semantic config).
3. `_ensure_knowledge_base(...)` seeds the Foundry IQ knowledge source +
   knowledge base (create-or-update; **no per-document work**).
4. Prints the AZURE_* summary.

No blob upload anywhere in `post_provision.py`. It is schema/KB bootstrap only —
so the survey's "post_provision.py does NOT upload sample data" is literally
true; its error was generalizing to "no upload step anywhere."

### 1.5 Verdict (Q3)

**After a fresh `azd up` in cosmosdb mode:**

- `postprovision` → index schema + Foundry IQ KB exist but are **empty**.
- `postdeploy` → uploads the default benefits PDFs + enqueues ingestion →
  Functions `batch_start`/`batch_push` index them → **chat grounds**.

So the chat WILL have grounded documents out-of-the-box UNLESS the operator
selects `0/skip` in the interactive menu or sets `AZURE_ENV_SAMPLE_DATA=none`.
"Schema-only + empty" is only the state *between* provision and deploy, or when
the seed is explicitly opted out.

---

## 2. `azd up` command flow + required env values (Q4)

### 2.1 azure.yaml wiring

- `name: chat-with-your-data-v2`, `requiredVersions.azd: ">= 1.18.0 != 1.23.9"`.
- `infra: { provider: bicep, path: infra, module: main }`.
- `services:` — three services, all `host: containerapp`:
  - `backend` → `./src/backend`, `docker.path: ../../docker/Dockerfile.backend`,
    `context: ../..`, `remoteBuild: true`.
  - `frontend` → `./src/frontend`, `docker.path:
    ../../docker/Dockerfile.frontend`, `context: ../..`, `target: prod`,
    `remoteBuild: true`.
  - `function` → `./src/functions`, `docker.path:
    ../../docker/Dockerfile.functions`, `context: ../..`, `remoteBuild: true`.
  - Services bind to Bicep resources by the `azd-service-name` tag
    (`backend` / `frontend` / `function`).
  - **`remoteBuild: true` on all three** → images build in ACR, not the local
    Docker daemon. A local Docker install is NOT required for `azd deploy` /
    `azd up`.
- **Typed prompts** (`parameters:` block) surfaced by `azd up` / `azd provision`:
  `databaseType` (default `cosmosdb`), `azureAiServiceLocation` (default
  `eastus2`), `enableMonitoring` / `enableScalability` / `enableRedundancy` /
  `enablePrivateNetworking` (all default `false`). azd persists each answer as
  `AZURE_ENV_<name>` and only prompts when the value is not already in the env.

### 2.2 Operator flow (cosmosdb, defaults)

```powershell
cd v2
azd auth login                        # or: azd auth login --check-status
azd env new <AZD_ENV_NAME>            # or: azd env select <AZD_ENV_NAME>
# (optional) pre-seed answers so azd doesn't prompt:
azd env set AZURE_ENV_DATABASE_TYPE cosmosdb
azd env set AZURE_ENV_AI_SERVICE_LOCATION eastus2
azd up                                # provision + deploy + postprovision + postdeploy
```

`azd up` prompt sequence on a fresh env (each only if unset): **subscription →
location (`AZURE_LOCATION`) → databaseType → azureAiServiceLocation →
enableMonitoring → enableScalability → enableRedundancy →
enablePrivateNetworking**.

### 2.3 Required env values before `azd up`

From `v2/infra/main.parameters.json` (azd token substitution) +
`v2/infra/main.bicep` (`@allowed` / defaults):

| azd env var | Bicep param | Default | Required? | Notes |
|---|---|---|---|---|
| `AZURE_LOCATION` | `location` | none | **YES** | azd standard location prompt. Bicep `@allowed`: `australiaeast`, `eastus2`, `japaneast`, `uksouth` (narrower than AI-service list). |
| `AZURE_ENV_AI_SERVICE_LOCATION` | `azureAiServiceLocation` | `${AZURE_LOCATION}` (param file) / `eastus2` (typed prompt) | effectively yes | Model-availability region; 11 allowed regions incl. `eastus2`. |
| `AZURE_ENV_DATABASE_TYPE` | `databaseType` | `cosmosdb` | no | `cosmosdb` \| `postgresql`. Locked after deploy. |
| `AZURE_ENV_INGESTION_TRIGGER` | `ingestionTrigger` | `direct_enqueue` | no | `direct_enqueue` \| `event_grid`. |
| `AZURE_PRINCIPAL_ID` | `createdBy` / `postgresAdminPrincipalId` | set by azd from login | auto | — |
| `AZURE_ENV_POSTGRES_ADMIN_PRINCIPAL_NAME` | `postgresAdminPrincipalName` | none | postgres only | Unused in cosmosdb mode. |
| `AZURE_CONTAINER_REGISTRY_ENDPOINT` | `backendContainerRegistryHostname` | empty | auto | Populated by azd after first provision. |
| `AZURE_ENV_IMAGE_TAG` | `backendContainerImageTag` | `latest` | no | See WI-02 (§6). |
| `AZURE_ENV_ENABLE_MONITORING/SCALABILITY/REDUNDANCY/PRIVATE_NETWORKING` | WAF flags | `false` | no | Typed prompts. |
| `AZURE_ENV_SAMPLE_DATA` | (postdeploy only) | unset → default seed | no | `default`\|`contract`\|`employee`\|`all`\|`none`. |

Model params also have azd-overridable defaults (all in `main.parameters.json`):
`gptModelName=gpt-5.1`, `gptModelVersion=2025-11-13`,
`gptModelDeploymentType=GlobalStandard`, `gptModelCapacity=150`,
`embeddingModelName=text-embedding-3-large`, `embeddingModelVersion=1`,
`embeddingModelDeploymentType=Standard`, `embeddingModelCapacity=100`.

---

## 3. what-if pre-check (Q5)

### 3.1 Deployment scope — CORRECTION

The task brief assumed a subscription-scoped what-if. Ground truth:

```bicep
// v2/infra/main.bicep line 25
targetScope = 'resourceGroup'
```

So the deployment is **resource-group scoped**, and the raw what-if is
`az deployment group what-if`, **not** `az deployment sub what-if`.

### 3.2 Recommended: azd-native what-if

Because `main.parameters.json` uses azd `${AZURE_ENV_*}` substitution tokens
that plain `az` will NOT expand, the clean pre-check is the azd-native preview
(it resolves the env, compiles Bicep, and runs the ARM what-if for you):

```powershell
cd v2
azd provision --preview
```

### 3.3 Raw `az` form (needs a pre-existing RG + resolved params)

If a planner wants the raw ARM what-if, the RG must already exist and the azd
substitution tokens must be resolved (via `azd env get-values`). Shape (tokens
only, never real IDs):

```powershell
# RG must exist first; azd normally creates it during provision.
az deployment group what-if `
  --resource-group <RESOURCE_GROUP> `
  --template-file v2/infra/main.bicep `
  --parameters location=<REGION> `
               azureAiServiceLocation=<REGION> `
               databaseType=cosmosdb `
               solutionName=<SUFFIX> `
               createdBy=<AZURE_PRINCIPAL_OBJECT_ID>
```

Note: passing `--parameters v2/infra/main.parameters.json` directly will fail on
the `${AZURE_ENV_*}` tokens — `az` does not do azd substitution. Prefer
`azd provision --preview`. Parameters file of record:
`v2/infra/main.parameters.json`.

---

## 4. azd version behavior (Q6 / WI-02)

- **Observed:** `azd version 1.27.0 (commit dcd7c53153…) (stable)` (ran
  `azd version`; first invocation printed nothing, re-ran with `2>&1 |
  Out-String`).
- **Pin `>= 1.18.0 != 1.23.9`:** `1.27.0 >= 1.18.0` ✓ and `1.27.0 != 1.23.9` ✓
  → **SATISFIED.**
- **WI-02 unique image tags:** For `host: containerapp` services, azd tags each
  build with a unique per-build tag (azd's default tag is
  `azd-deploy-<unix-timestamp>`), then updates the Container App to that exact
  tag — it does not rely on a floating `:latest`. So every `azd deploy` /
  `azd up` rolls fresh code and forces a new Container App revision. (The Bicep
  `backendContainerImageTag` default `latest` is only the *initial* image the
  Container App is created with at provision time; `azd deploy` overrides it with
  the unique per-build tag.) azd `1.27.0` is well past `1.18.0`, so this
  unique-tag behavior is present. WI-02 is satisfied by the installed version.

---

## 5. Local docker build pre-check (Q7)

Dockerfiles confirmed present: `v2/docker/Dockerfile.backend`,
`Dockerfile.frontend`, `Dockerfile.functions`. From `v2/azure.yaml`, each
service's docker `context: ../..` (relative to `./src/<svc>`) resolves to `v2/`,
and `path: ../../docker/Dockerfile.<svc>` resolves to `v2/docker/Dockerfile.<svc>`.

Run from the `v2/` directory (build context = `.` = `v2/`):

```powershell
# Backend
docker build -f docker/Dockerfile.backend -t cwyd-backend:local .

# Frontend (pin the multi-stage build to the `prod` stage, per azure.yaml)
docker build -f docker/Dockerfile.frontend --target prod -t cwyd-frontend:local .

# Function
docker build -f docker/Dockerfile.functions -t cwyd-function:local .
```

- The **frontend build will FAIL until WI-07 (TS6133) is fixed** — the `prod`
  stage runs the TypeScript/Vite build under strict unused-symbol checking, and
  a TS6133 "declared but never used" error aborts the build. (Relayed per brief;
  not independently reproduced in this read-only pass.)
- `remoteBuild: true` means `azd deploy` builds in ACR regardless, so a broken
  local build does not block ACR builds — but a local `docker build` is the
  fastest pre-check and will surface WI-07 immediately.

---

## 6. Post-deploy validation (Q8)

### 6.1 Health endpoints

`v2/src/backend/routers/health.py` (prefix `/api`):

- `GET /api/health` — **always HTTP 200**; severity is in the body `status`
  field (`pass` / `degraded` / `fail`). Diagnostic-friendly.
- `GET /api/health/ready` — readiness probe; returns **HTTP 503** when overall
  status is `FAIL`, else 200.

Checks (shallow config/construction only) — `v2/src/backend/services/health.py`
`run_health_checks(settings)` runs three `DependencyCheck`s:

- `_check_foundry` → FAIL if `AZURE_AI_PROJECT_ENDPOINT` or
  `AZURE_OPENAI_GPT_DEPLOYMENT` unset.
- `_check_database` → FAIL if no endpoint for the configured `db_type`
  (cosmos_endpoint in cosmosdb mode).
- `_check_search` → FAIL if AzureSearch mode and `AZURE_AI_SEARCH_ENDPOINT`
  unset; `SKIP` in pgvector mode (skip is neutral, never drags overall down).

Aggregation: any FAIL → overall FAIL; else PASS. Deep round-trip liveness is
explicitly deferred (docstring: "Phase 6").

### 6.2 Confirm each Container App pulled the new image

The `cloud_deployment.md` runbook (dated 2026-06-05) still describes frontend as
App Service / function as Function App — that runbook **predates** the
containerapp conversion the brief describes, so for the *current* all-Container-App
topology use Container App revision checks for all three:

```powershell
az containerapp revision list -g <RESOURCE_GROUP> -n ca-backend-<SUFFIX>  -o table
az containerapp revision list -g <RESOURCE_GROUP> -n ca-frontend-<SUFFIX> -o table
az containerapp revision list -g <RESOURCE_GROUP> -n ca-function-<SUFFIX> -o table
# Expect the newest revision Active=True, TrafficWeight=100, and its image tag
# matching the azd per-build tag from the deploy output.
```

Image-tag confirmation per app:

```powershell
az containerapp show -g <RESOURCE_GROUP> -n ca-backend-<SUFFIX> `
  --query "properties.template.containers[0].image" -o tsv
```

(Resource names are placeholders — confirm actual names via `azd env get-values`
after deploy; the runbook's `ca-backend-<SUFFIX>` etc. are illustrative.)

### 6.3 Minimal chat smoke test

Endpoint: `POST /api/conversation` (`v2/src/backend/routers/conversation.py`,
`router = APIRouter(prefix="/api")`, `@router.post("/conversation")`). Body is a
`ConversationRequest` (messages + conversation_id). Example (from the runbook):

```powershell
$body = '{"messages":[{"role":"user","content":"hello"}],"conversation_id":"smoke-001"}'
Invoke-RestMethod -Uri "https://<BACKEND_FQDN>/api/conversation" `
  -Method POST -ContentType "application/json" -Body $body
```

- First call on the `agent_framework` branch takes ~3–5 s (lazy Foundry
  `create_agent` + Cosmos `upsert_agent_id`); subsequent calls hit the cache.
- SSE channels to expect: `reasoning`, `tool`, `answer`, `citation`, `error`.
- **Grounding caveat:** a doc-specific question returns "no grounding" / the
  fixed out-of-domain fallback **until documents are indexed**. With the
  `postdeploy` seed (§1) the default benefits PDFs are already indexed, so a
  benefits question should return a grounded answer with a `citation` event. If
  the seed was skipped, expect no grounding until an operator uploads docs.

OpenAPI cross-check: `GET /openapi.json` should list `/api/conversation`,
`/api/admin/config`, `/api/admin/documents`, `/api/history/conversations`,
`/api/health`.

---

## 7. Quota pre-check (Q9 / WI-01)

### 7.1 The two model deployments the Bicep creates

`v2/infra/main.bicep` (deployments block, ~line 539) + defaults (~line 142-174)
+ `v2/infra/main.parameters.json`:

| Role | Model name | Version | SKU (deploymentType) | Capacity | Quota usageName token (bicep `@metadata`) |
|---|---|---|---|---|---|
| Chat | `gpt-5.1` | `2025-11-13` | `GlobalStandard` | `150` | `OpenAI.GlobalStandard.gpt-5.1,150` |
| Embedding | `text-embedding-3-large` | `1` | `Standard` | `100` | `OpenAI.Standard.text-embedding-3-large,100` |

The `azureAiServiceLocation` param carries the azd usageName hints:

```bicep
usageName: [
  'OpenAI.GlobalStandard.gpt-5.1,150'
  'OpenAI.Standard.text-embedding-3-large,100'
]
```

RAI policy `Microsoft.DefaultV2` is attached to the chat deployment.

### 7.2 Read-only quota verification command shape

```powershell
# GlobalStandard chat quota in the AI-service region (need >= 150 for gpt-5.1)
az cognitiveservices usage list --location <REGION> `
  --query "[?contains(name.value, 'OpenAI.GlobalStandard.gpt-5.1')].{name:name.value, current:currentValue, limit:limit}" `
  -o table

# Standard embedding quota (need >= 100 for text-embedding-3-large)
az cognitiveservices usage list --location <REGION> `
  --query "[?contains(name.value, 'OpenAI.Standard.text-embedding-3-large')].{name:name.value, current:currentValue, limit:limit}" `
  -o table
```

Check `limit - current >= required` before provision. `<REGION>` =
`azureAiServiceLocation` (default `eastus2`). Capacities are in 1000-TPM units
(150 = 150K TPM chat, 100 = 100K TPM embedding).

---

## 8. Silent-except gate (Q10)

### 8.1 The offending code

`v2/src/functions/core/search_resolution.py`, `resolve_search_provider`
(lines ~85-99):

```python
    provider: BaseSearch | None = None
    try:
        if search_key == IndexStore.PGVECTOR:
            pool_helper = PgVectorPool(settings=settings, credential=credential)
            search_kwargs["pool"] = await pool_helper.acquire()
        provider = search_registry.registry.get(search_key)(**search_kwargs)
        await provider.ensure_schema()
    except BaseException:            # <-- line ~91: flagged construct
        if provider is not None:
            await provider.aclose()
        if pool_helper is not None:
            await pool_helper.aclose()
        raise                        # <-- re-raises; NOT a silent swallow
```

This is a **cleanup-and-reraise** — it does NOT silently swallow. But the gate
bans `except BaseException` **unconditionally**.

### 8.2 Why the gate flags it even though it re-raises

`v2/tests/test_no_silent_excepts.py` enforces two separate rules; rule 1 is
target-based, independent of the body:

> "1. **No `except BaseException`.** Catches `KeyboardInterrupt` / `SystemExit`
> … Always wrong."

The detector's own self-check fixture proves it flags the re-raising form:

```python
("try:\n    x = 1\nexcept BaseException:\n    raise\n", {"except BaseException"}),
```

So `search_resolution.py` line ~91 IS reported as `except BaseException`. The
`_EXEMPTIONS` set is `frozenset()` (empty by design) — there is no per-file
escape hatch; the fix is to narrow the catch to `except Exception:` (or the SDK
umbrella) while keeping the cleanup/re-raise.

### 8.3 Is it a hard gate? Does it block deploy?

- **Hard pytest gate?** Yes for a green test lane — `test_no_silent_excepts` is
  parametrized per-file under `v2/src/**`, so this file produces a **failing**
  test case. A CI/local lane that runs `uv run pytest` will go red.
- **Deploy blocker?** **No.** `azd up` / `azd provision` / `azd deploy` never run
  pytest — they compile Bicep, provision, and build/push container images. The
  red test does not stop a deploy. It only blocks a green pytest/CI run and any
  "all tests pass" preflight gate the operator chooses to enforce.
- **Ownership:** the file docstring declares `Phase: 6 (Functions blueprints /
  modular RAG indexing pipeline)`. Per the repo's no-mid-phase-backfill rule
  (Hard Rule #12) and the brief, this belongs to the separate Phase 6 effort —
  it is a red test to be fixed by that effort, not a first-`azd up` blocker.

---

## 9. Cross-check against prior tracking notes

| Prior claim | Source | Verdict |
|---|---|---|
| "post_provision.py does NOT upload sample data" | 2026-06-25 infra survey | TRUE (about that script) |
| "no sample-data upload step anywhere in the v2 azd flow" | 2026-06-25 infra survey | **FALSE / STALE** — the `postdeploy` hook uploads via `upload_sample_data.py` |
| "no v2/data/ folder exists" | 2026-06-25 infra survey | TRUE — sample data is repo-root `data/` |
| "the `postdeploy` upload exists; verify it grounds" | WI-03 planning log | TRUE — confirmed |

---

## 10. Deploy readiness — net assessment

### Ordered blockers before a clean first `azd up`

1. **WI-01 (quota) — gating.** Verify GlobalStandard `gpt-5.1` >= 150 and
   Standard `text-embedding-3-large` >= 100 in `<REGION>` (§7). A quota shortfall
   fails provision at the model-deployment step.
2. **WI-07 (frontend TS6133) — gating for a local build; NOT gating for azd.**
   `remoteBuild: true` builds the frontend in ACR, so `azd deploy` can still
   push — but the same TS6133 will fail the ACR build too if the error is in the
   `prod` stage. Fix before relying on the frontend image. Confirm whether the
   ACR build hits the same tsc step (very likely).
3. **azd env values set** — `AZURE_LOCATION` (one of the 4 allowed regions),
   `AZURE_ENV_AI_SERVICE_LOCATION`, `AZURE_ENV_DATABASE_TYPE=cosmosdb` (§2.3).
4. **Auth + subscription** — `azd auth login`; correct subscription selected.
5. **(Recommended) `azd provision --preview`** what-if to catch RBAC / region /
   capacity drift before the real run (§3).

### Non-blockers (do not stop `azd up`)

- **Silent-except gate** (`test_no_silent_excepts` on `search_resolution.py`) —
  red pytest, Phase 6 ownership, NOT a deploy blocker (§8).
- **`cloud_deployment.md` hosting-model drift** — the runbook still names
  frontend=App Service / function=Function App; the current topology is
  all-Container-Apps. Deploy commands (`azd up`, `azd deploy <svc>`) and smoke
  matrix remain valid; only the revision-check resource types shift to
  `az containerapp` (§6.2).

### Resolved sample-data grounding truth

**A fresh `azd up` in cosmosdb mode GROUNDS out-of-the-box.** The `postdeploy`
hook (`upload_sample_data.py`) uploads the default benefits PDFs from repo-root
`data/` into the `documents` container and enqueues ingestion; the Functions
pipeline indexes them into the `cwyd-index` / Foundry IQ KB that `postprovision`
created. The chat is grounded unless the operator picks `skip`/`0` in the menu
or sets `AZURE_ENV_SAMPLE_DATA=none`. The index/KB are schema-only + empty ONLY
in the window between provision and deploy, or when the seed is opted out.

### tooling snapshot

- `azd version` = **1.27.0** — satisfies `>= 1.18.0 != 1.23.9`.
- Deployment scope = **resourceGroup** (`az deployment group what-if` /
  `azd provision --preview`, not subscription).

---

## Appendix: evidence index (file:line)

- `v2/azure.yaml` — services (all `host: containerapp`, `remoteBuild: true`),
  typed `parameters:` prompts, `hooks.postprovision` + `hooks.postdeploy`.
- `v2/scripts/upload_sample_data.py` — `_curated_data_dir()` (repo-root `data/`),
  `upload_blob_if_absent`, `enqueue_ingest_message`, `resolve_selection`,
  `_BENEFITS_SET`, 300s index-completion poll.
- `v2/scripts/upload-sample-data.ps1` — pwsh shim → `uv run python
  upload_sample_data.py`.
- `v2/scripts/post_provision.py` — index schema + Foundry IQ KB only (no upload).
- `v2/infra/main.bicep:25` — `targetScope = 'resourceGroup'`; `:142-174` model
  params; `:73-88` `azureAiServiceLocation` usageName; `:539-566` deployments
  block.
- `v2/infra/main.parameters.json` — azd token bindings + model defaults.
- `v2/src/backend/routers/health.py` — `/api/health`, `/api/health/ready`.
- `v2/src/backend/services/health.py` — `run_health_checks` (foundry/database/search).
- `v2/src/backend/routers/conversation.py:57-60` — `POST /api/conversation`.
- `v2/tests/test_no_silent_excepts.py` — gate rules + self-check fixture.
- `v2/src/functions/core/search_resolution.py:~91` — `except BaseException:` (re-raises).
- `v2/docker/` — `Dockerfile.backend`, `Dockerfile.frontend`, `Dockerfile.functions`.
- `v2/docs/cloud_deployment.md` — deploy runbook (stale hosting model; valid commands).
- `azd version` → `1.27.0`.
