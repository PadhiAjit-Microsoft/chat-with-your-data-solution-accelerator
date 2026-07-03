# BUG-0055 — App Insights zero-telemetry: Bicep infra wiring audit

Status: Complete (read-only research)
Date: 2026-07-02
Scope: `v2/infra/**` Bicep only. No files modified outside this research doc.
All real Azure IDs/suffixes replaced with `<SUFFIX>` / `<DATA_SUFFIX>` placeholders.

## TL;DR answers to the return questions

- `APPLICATIONINSIGHTS_CONNECTION_STRING` injected into **backend** (`ca-backend-<SUFFIX>`)? **NO — not under that name (by design).** The backend gets a *differently named* env var `AZURE_APP_INSIGHTS_CONNECTION_STRING` (main.bicep line 1911), NOT the standard `APPLICATIONINSIGHTS_CONNECTION_STRING`. Both source the same value. This is the deliberate Amendment-1 fix (see section b).
- `APPLICATIONINSIGHTS_CONNECTION_STRING` injected into **functions** (`ca-func-<SUFFIX>`)? **YES** — standard name, main.bicep line 2191.
- `AZURE_ENVIRONMENT` wired on both? **YES = `production`** on both (backend line 1804, functions line 2159). BUG-0069's missing-`AZURE_ENVIRONMENT` failure mode does NOT apply here.
- Local auth disabled on App Insights? **YES — `disableLocalAuth: true`** (main.bicep line 326). Entra-authenticated ingestion is the ONLY accepted path. `Monitoring Metrics Publisher` role IS granted to the workload UAMI (lines 330-336).
- Single most likely infra root cause: **`disableLocalAuth: true` forces Entra-token ingestion, but neither runtime is guaranteed by infra to actually present an AAD token to the ingestion endpoint** — the Functions host has no `APPLICATIONINSIGHTS_AUTHENTICATION_STRING`, and the Python exporters (per bugs.md) are invoked as `configure_azure_monitor(connection_string=...)` with no `credential=`. Connection-string-only auth against a `disableLocalAuth: true` component returns a silent 401 → "zero telemetry ever from BOTH." (Full ranking in section g.)

---

## (a) App Insights resource declaration + outputs

File: `v2/infra/main.bicep`

- **Log Analytics workspace** (the sink) — lines 304-314:
  - AVM module `br/public:avm/res/operational-insights/workspace:0.11.2`, gated `if (enableMonitoring)`.
  - `name: 'log-<SUFFIX>'`, `skuName: 'PerGB2018'`, `dataRetention: 90|30`.
- **Application Insights** — lines 316-339:
  - AVM module `br/public:avm/res/insights/component:0.6.0`, gated `if (enableMonitoring)`.
  - `name: 'appi-<SUFFIX>'` (line 319).
  - `workspaceResourceId: logAnalyticsWorkspace!.outputs.resourceId` (line 323) — **workspace-based**, correctly wired to `log-<SUFFIX>`. This path is confirmed working: bugs.md notes the host emits ~500 `Information` rows/24h to `log-<SUFFIX>` `FunctionAppLogs` via the `allLogs` diagnostic setting, so the App Insights → Log Analytics workspace link and the LAW itself are healthy.
  - `applicationType: 'web'` (line 324), `kind: 'web'` (line 325).
  - `disableLocalAuth: true` (line 326) — **the pivotal setting; see section e.**
  - `roleAssignments` (lines 330-336): grants **`Monitoring Metrics Publisher`** to `userAssignedIdentity.outputs.principalId` (`principalType: 'ServicePrincipal'`) scoped to the AppI component.
- **Output** — line 2569:
  - `output AZURE_APP_INSIGHTS_CONNECTION_STRING string = enableMonitoring ? applicationInsights!.outputs.connectionString : ''`
  - The Bicep surfaces only `connectionString`. No `instrumentationKey` output. All three consumers (backend env, functions env, template output) read `applicationInsights!.outputs.connectionString` (lines 1915, 2192, 2569).
- **`enableMonitoring`** — declared `param enableMonitoring bool = true` (line 208). Default-on per ADR-0018. Because `appi-<SUFFIX>` exists in the failing environment, `enableMonitoring` is effectively true, so both connection-string env vars ARE injected (the module and both env blocks share the same gate).

## (b) Backend container app — is the connection string env var wired?

Container app: `ca-backend-<SUFFIX>` (var `backendAppName = 'ca-backend-${solutionSuffix}'`, line 1552).
Declared as AVM module `backendContainerApp` → `br/public:avm/res/app/container-app:0.22.1` (line 1722).

**Wired — but under a NON-standard name, on purpose.** main.bicep lines 1904-1918:

```
enableMonitoring
  ? [
      {
        // Backend ACA ... calls configure_azure_monitor with the connection
        // string read from the AZURE_-prefixed typed setting
        // (ObservabilitySettings, env_prefix=AZURE_). ... (ADR-0018, BUG-0055.)
        name: 'AZURE_APP_INSIGHTS_CONNECTION_STRING'          // line 1911
        value: applicationInsights!.outputs.connectionString  // line 1915
      }
    ]
  : []
```

- The backend does **NOT** receive `APPLICATIONINSIGHTS_CONNECTION_STRING`. It receives `AZURE_APP_INSIGHTS_CONNECTION_STRING`.
- Rationale (ADR-0018 Amendment 1, 2026-06-23; bugs.md BUG-0055 "Backend half"): the backend ACA container has no host-level App Insights agent. Its Python lifespan reads the connection string from `ObservabilitySettings`, which sets `env_prefix="AZURE_"`, so the field only binds `AZURE_APP_INSIGHTS_CONNECTION_STRING`. The original Bicep wired the standard name, so the typed setting stayed empty and `configure_azure_monitor` never fired → the *original* backend zero-telemetry cause. The Bicep now sets the correct name.
- **Deploy-state caveat (critical for the backend half):** this rename is durable in Bicep but "takes effect on the next `azd provision`" (ADR-0018 / bugs.md). Env-var changes on a Container App are applied by `azd provision` (template redeploy), NOT by `azd deploy` (image push). BUG-0058's 2026-07-02 live confirmation was an `azd deploy function` (image), which does not re-apply backend env vars. **If `azd provision` has not run against this environment since 2026-06-23, the live `ca-backend-<SUFFIX>` revision still carries the old `APPLICATIONINSIGHTS_CONNECTION_STRING` name and the typed setting is still empty → guaranteed zero backend telemetry regardless of auth.**

## (c) Functions container app — is the connection string env var wired?

Container app: `ca-func-<SUFFIX>` (var `functionContainerAppName = 'ca-func-${solutionSuffix}'`, line 1554).
Declared as a **raw resource** `functionContainerApp 'Microsoft.App/containerApps@2024-10-02-preview'` (line 2078), `kind: 'functionapp'`, `identity: UserAssigned` (the shared UAMI). Authored raw (not the AVM module) because the pinned module version does not expose `kind=functionapp`.

**Wired — standard name.** main.bicep lines 2188-2196:

```
enableMonitoring
  ? [
      {
        name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'         // line 2191
        value: applicationInsights!.outputs.connectionString  // line 2192
      }
    ]
  : []
```

- The Functions host reads `APPLICATIONINSIGHTS_CONNECTION_STRING` natively — correct standard name.
- bugs.md confirms this value **matches** the `appi-<SUFFIX>` ingestion endpoint + ikey exactly (verified), so it is NOT a connection-string mismatch.
- **Infra gap:** there is **no `APPLICATIONINSIGHTS_AUTHENTICATION_STRING`** env var anywhere in the function container (or the whole tree — see section e). With `disableLocalAuth: true`, the Functions host's native App Insights integration cannot switch to Entra-token ingestion without it, so host-emitted `requests`/dependency telemetry silently 401s. (bugs.md already flags host `requests` telemetry as a possible separate follow-up.)
- The function **worker** (Python OTel) is a separate path: bugs.md "Function half" added `configure_telemetry()` in `src/functions/core/telemetry.py` calling `configure_azure_monitor(connection_string=...)`. See section e for why `connection_string=` alone still 401s under `disableLocalAuth: true`.

## (d) AZURE_ENVIRONMENT wiring on both

- Backend: `{ name: 'AZURE_ENVIRONMENT', value: 'production' }` — main.bicep line 1804.
- Functions: `{ name: 'AZURE_ENVIRONMENT', value: 'production' }` — main.bicep line 2159.

Both are hard-pinned to `production`. This is the correct dev-default-flipped-by-IaC pattern. **BUG-0069's "Bicep never wired AZURE_ENVIRONMENT → runtime reports environment=local" failure mode is NOT present here.** If the app gates telemetry on `environment == production`, that gate is satisfied on both runtimes. AZURE_ENVIRONMENT is not a BUG-0055 root cause.

## (e) Ingestion auth — DisableLocalAuth / Monitoring Metrics Publisher / AMPLS

- **`disableLocalAuth: true`** on the AppI component — main.bicep line 326. This is the WAF-aligned posture (ADR-0018): the ingestion data plane **refuses instrumentation-key / connection-string-only auth and requires a Microsoft Entra bearer token**.
- **`Monitoring Metrics Publisher`** role — granted to the workload UAMI on the AppI scope (main.bicep lines 330-336). This is the correct built-in role (`3913510d-42f4-4e42-8a64-81b1edca285c`, `Microsoft.Insights/telemetry/write`) for AAD-authenticated ingestion. The RBAC side is correct and present.
- **`APPLICATIONINSIGHTS_AUTHENTICATION_STRING` — ABSENT.** A `grep` for `AUTHENTICATION_STRING` / `Authorization=AAD` across `v2/infra/**` returns zero hits in any container env block. For a Functions **host** to use managed-identity ingestion against a `disableLocalAuth: true` component, it must be given `APPLICATIONINSIGHTS_AUTHENTICATION_STRING = "Authorization=AAD;ClientId=<uami-clientId>"`. Its absence means the host cannot present a token → 401.
- **Python exporter credential — not expressible in infra, and per bugs.md not passed.** `azure-monitor-opentelemetry`'s `configure_azure_monitor()` only sends an Entra token if called with `credential=` (e.g. `ManagedIdentityCredential(client_id=AZURE_CLIENT_ID)`). bugs.md documents BOTH the backend fix and the function-worker fix as `configure_azure_monitor(connection_string=...)` — **no `credential=` shown**. Against `disableLocalAuth: true`, connection-string-only ingestion = silent 401. This is the auth mechanism that reconciles "connection string is correct AND the exporter was fixed, yet still zero telemetry."
- **AMPLS / private link:** no `Microsoft.Insights/privateLinkScopes`, no `ambientLinkScope`, no AMPLS resource found in `v2/infra/**`. In the private-networking profile (`enablePrivateNetworking`) the AppI/LAW ingestion endpoints are public with no private-link scope. Not the current root cause (the environment reaches the ingestion endpoint enough to 401, and LAW `allLogs` works), but noted for completeness.

## (f) Full parameter-flow chain: AppI output → env var

Single, clean chain (no break in the value plumbing):

```
logAnalyticsWorkspace (module, line 304)
    └─ outputs.resourceId
         └─► applicationInsights.params.workspaceResourceId  (line 323)

applicationInsights (module, line 316)
    └─ outputs.connectionString
         ├─► backend  env: AZURE_APP_INSIGHTS_CONNECTION_STRING = applicationInsights!.outputs.connectionString  (lines 1911/1915)
         ├─► function env: APPLICATIONINSIGHTS_CONNECTION_STRING  = applicationInsights!.outputs.connectionString  (lines 2191/2192)
         └─► template output AZURE_APP_INSIGHTS_CONNECTION_STRING = applicationInsights!.outputs.connectionString  (line 2569)

userAssignedIdentity (module, line 288)
    └─ outputs.principalId
         └─► applicationInsights.params.roleAssignments[0].principalId  ("Monitoring Metrics Publisher", lines 330-336)
```

- The value chain (connection string → both env vars) is **intact and direct** — no missing hop, no wrong variable, no cross-RG patch. The App Insights and Log Analytics resources co-locate in the workload RG.
- The chain that is **NOT** present is the *auth* chain for the ingestion call: nothing wires `APPLICATIONINSIGHTS_AUTHENTICATION_STRING` (functions host) and nothing in infra can make the Python exporter pass `credential=`. `disableLocalAuth: true` makes that auth chain mandatory, and it is the part that breaks.

## (g) Ranked assessment — infra root cause

Because the connection-string value chain is intact and `AZURE_ENVIRONMENT` is correct on both, the surviving infra-side causes are all downstream of `disableLocalAuth: true`.

1. **[Most likely, affects BOTH] `disableLocalAuth: true` with no Entra-token ingestion actually wired.** The component refuses ikey/connection-string-only auth (line 326). The `Monitoring Metrics Publisher` grant exists, but presenting the token is the caller's job and neither caller is set up for it: the Functions host lacks `APPLICATIONINSIGHTS_AUTHENTICATION_STRING`, and the Python exporters (backend + function worker) are called `configure_azure_monitor(connection_string=...)` with no `credential=` (per bugs.md). Result: both runtimes' application telemetry silently 401s → "zero telemetry ever from BOTH," exactly the symptom. This is the single cause that explains both halves simultaneously and survives the documented env-var-name + worker-OTel fixes.

2. **[Backend-specific deploy-state] Env has not been `azd provision`-ed since the 2026-06-23 backend rename.** The backend env var is now `AZURE_APP_INSIGHTS_CONNECTION_STRING` in Bicep, but it only reaches the live revision via `azd provision`, not `azd deploy`. If the last provision predates the Amendment-1 fix, the running `ca-backend-<SUFFIX>` still has the old standard name → typed setting empty → zero backend telemetry independent of auth. (Verify with `az containerapp show -g <RESOURCE_GROUP> -n ca-backend-<SUFFIX>` and check the env var name.)

3. **[Functions host `requests` only] Missing `APPLICATIONINSIGHTS_AUTHENTICATION_STRING`.** Even if the Python worker OTel is fixed, host-emitted invocation (`requests`) telemetry needs the AAD auth-string on `ca-func-<SUFFIX>`. Its absence keeps host-level `requests` empty while worker `traces` might appear — a partial-telemetry outcome. (bugs.md pre-flags this as a `host.json` `telemetryMode` follow-up.)

4. **[Not a cause — ruled out]** Connection-string mismatch (verified matching), AppI↔LAW wiring (LAW `allLogs` works), `AZURE_ENVIRONMENT` (correctly `production` on both), `enableMonitoring` gate (appi exists ⇒ true ⇒ env vars injected), AMPLS/private-link (none configured).

**Net infra verdict:** The Bicep is NOT missing the connection-string env var for either app, and the value plumbing is correct. The infra decision that most plausibly explains the total zero-telemetry outcome is `disableLocalAuth: true` (line 326) *without* the corresponding Entra-token ingestion wiring — no Functions-host `APPLICATIONINSIGHTS_AUTHENTICATION_STRING` and Python exporters invoked without a `credential`. The cheapest infra-only levers to disambiguate/fix: (i) add `APPLICATIONINSIGHTS_AUTHENTICATION_STRING = "Authorization=AAD;ClientId=<uamiClientId>"` to the function env (host path); (ii) confirm the backend/worker `configure_azure_monitor` calls pass a managed-identity credential (code path); or (iii) as a diagnostic isolation step only, temporarily set `disableLocalAuth: false` to confirm whether ikey-auth telemetry then flows (would prove the auth hypothesis) — not a durable fix given the WAF posture.

---

## Evidence index (file + line)

- `v2/infra/main.bicep`
  - 208 — `param enableMonitoring bool = true`
  - 304-314 — Log Analytics workspace module (`log-<SUFFIX>`)
  - 316-339 — App Insights module (`appi-<SUFFIX>`); 323 workspaceResourceId; 324 applicationType 'web'; 325 kind 'web'; 326 `disableLocalAuth: true`; 330-336 `Monitoring Metrics Publisher` → UAMI
  - 1552 — `var backendAppName = 'ca-backend-${solutionSuffix}'`
  - 1554 — `var functionContainerAppName = 'ca-func-${solutionSuffix}'`
  - 1722 — `module backendContainerApp 'br/public:avm/res/app/container-app:0.22.1'`
  - 1804 — backend `AZURE_ENVIRONMENT = 'production'`
  - 1904-1918 — backend App Insights env block; 1911 name `AZURE_APP_INSIGHTS_CONNECTION_STRING`; 1915 value `applicationInsights!.outputs.connectionString`
  - 2078 — `resource functionContainerApp 'Microsoft.App/containerApps@2024-10-02-preview'` (kind `functionapp`)
  - 2159 — functions `AZURE_ENVIRONMENT = 'production'`
  - 2188-2196 — functions App Insights env block; 2191 name `APPLICATIONINSIGHTS_CONNECTION_STRING`; 2192 value `applicationInsights!.outputs.connectionString`
  - 2569 — `output AZURE_APP_INSIGHTS_CONNECTION_STRING`
- `v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md` — default-on monitoring, UAMI `Monitoring Metrics Publisher`, `disableLocalAuth: true` intent; Amendment 1 (per-workload env-var names, BUG-0055 backend half).
- `v2/docs/bugs.md` — BUG-0055 detail (line ~998): connection string verified matching; LAW `allLogs` works; backend fix = env-var rename (durable in Bicep, needs `azd provision`); function fix = worker OTel `configure_azure_monitor(connection_string=...)`; both "not yet cloud-verified." BUG-0058 resolution (2026-07-02): function is now a Container App, `azd deploy function` live-confirmed — but that is `deploy`, not `provision`.

## Full env-var arrays (question 7)

### Backend `ca-backend-<SUFFIX>` (main.bicep ~1789-1918)
Base array: `AZURE_CLIENT_ID`, `AZURE_UAMI_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_ENVIRONMENT=production`, `AZURE_REQUIRE_ADMIN_AUTH=false`, `AZURE_AI_PROJECT_ENDPOINT`, `AZURE_OPENAI_ENDPOINT`, `AZURE_AI_SERVICES_ENDPOINT`, `AZURE_OPENAI_API_VERSION`, `AZURE_AI_AGENT_API_VERSION`, `AZURE_OPENAI_GPT_DEPLOYMENT`, `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`, `AZURE_DB_TYPE`, `AZURE_INDEX_STORE`, `AZURE_COSMOS_ENDPOINT`, `AZURE_AI_SEARCH_ENDPOINT`, `AZURE_AI_SEARCH_INDEX`, `AZURE_AI_SEARCH_KNOWLEDGE_BASE_NAME`, `AZURE_AI_SEARCH_KNOWLEDGE_SOURCE_NAME`, `AZURE_AI_SEARCH_KNOWLEDGE_BASE_API_VERSION`, `AZURE_AI_SEARCH_CONNECTION_NAME`, `AZURE_POSTGRES_ENDPOINT`, `AZURE_POSTGRES_ADMIN_PRINCIPAL_NAME`, `AZURE_SPEECH_SERVICE_NAME`, `AZURE_SPEECH_SERVICE_REGION`, `AZURE_SPEECH_ACCOUNT_RESOURCE_ID`, `AZURE_CONTENT_SAFETY_ENABLED=true`, `AZURE_CONTENT_SAFETY_ENDPOINT`, `CWYD_ORCHESTRATOR_NAME`, `AZURE_STORAGE_ACCOUNT_NAME`, `AZURE_DOCUMENTS_CONTAINER`, `AZURE_DOC_PROCESSING_QUEUE`, `AZURE_INGESTION_TRIGGER`, `BACKEND_CORS_ORIGINS`.
Monitoring union (when `enableMonitoring`): **`AZURE_APP_INSIGHTS_CONNECTION_STRING`** (NOT the standard name).
Absent: `APPLICATIONINSIGHTS_CONNECTION_STRING`, `APPLICATIONINSIGHTS_AUTHENTICATION_STRING`, `APPINSIGHTS_INSTRUMENTATIONKEY`.

### Functions `ca-func-<SUFFIX>` (main.bicep ~2118-2196)
Base array: `AzureWebJobsStorage__accountName`, `AzureWebJobsStorage__credential=managedidentity`, `AzureWebJobsStorage__clientId`, `FUNCTIONS_WORKER_RUNTIME=python`, `FUNCTIONS_EXTENSION_VERSION=~4`, `AZURE_CLIENT_ID`, `AZURE_UAMI_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_ENVIRONMENT=production`, `AZURE_AI_PROJECT_ENDPOINT`, `AZURE_OPENAI_ENDPOINT`, `AZURE_AI_SERVICES_ENDPOINT`, `AZURE_OPENAI_API_VERSION`, `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`, `AZURE_DB_TYPE`, `AZURE_INDEX_STORE`, `AZURE_COSMOS_ENDPOINT`, `AZURE_AI_SEARCH_ENDPOINT`, `AZURE_POSTGRES_ENDPOINT`, `AZURE_POSTGRES_ADMIN_PRINCIPAL_NAME`, `AZURE_STORAGE_ACCOUNT_NAME`, `AZURE_DOCUMENTS_CONTAINER`, `AZURE_DOC_PROCESSING_QUEUE`.
Monitoring union (when `enableMonitoring`): **`APPLICATIONINSIGHTS_CONNECTION_STRING`** (standard name).
Absent: `AZURE_APP_INSIGHTS_CONNECTION_STRING`, **`APPLICATIONINSIGHTS_AUTHENTICATION_STRING`** (the AAD-ingestion hint the host needs under `disableLocalAuth: true`), `APPINSIGHTS_INSTRUMENTATIONKEY`.

## Clarifying questions (cannot be answered from infra code alone)

1. Has `azd provision` (not just `azd deploy`) run against this environment since 2026-06-23? If not, ranked cause #2 (stale backend env-var name) is live and is the backend's actual root cause. Verify: `az containerapp show -g <RESOURCE_GROUP> -n ca-backend-<SUFFIX> --query "properties.template.containers[0].env[?name=='AZURE_APP_INSIGHTS_CONNECTION_STRING']"`.
2. Do the backend `configure_azure_monitor` (backend/app.py lifespan) and function worker `src/functions/core/telemetry.py` calls pass a `credential=` (ManagedIdentityCredential)? bugs.md shows only `connection_string=`. If no credential is passed, ranked cause #1 (silent 401 under `disableLocalAuth: true`) is confirmed. (Code question — outside this infra scope; flagged for the code-audit subagent.)
3. Is host-level invocation telemetry (`requests`) required, or is worker-level (`traces`/`dependencies`) sufficient? If `requests` is required, `APPLICATIONINSIGHTS_AUTHENTICATION_STRING` on `ca-func-<SUFFIX>` and/or `host.json` `telemetryMode` must be added.

## Recommended next research (not done this session)

- [ ] Code audit: confirm whether backend `configure_azure_monitor` and `functions/core/telemetry.py` pass a managed-identity `credential`; if not, that is the concrete fix for cause #1.
- [ ] Live check: read the deployed `ca-backend-<SUFFIX>` + `ca-func-<SUFFIX>` env vars via `az containerapp show` to confirm which connection-string names are actually present on the running revisions (settles cause #2).
- [ ] Live check: query the AppI ingestion endpoint / `az monitor app-insights` for 401 ingestion failures, or temporarily flip `disableLocalAuth: false` in a scratch deploy to confirm the auth hypothesis.
- [ ] Evaluate adding `APPLICATIONINSIGHTS_AUTHENTICATION_STRING = "Authorization=AAD;ClientId=<uamiClientId>"` to the function env block (host-path fix), and whether `host.json` `telemetryMode` is needed for host `requests`.
