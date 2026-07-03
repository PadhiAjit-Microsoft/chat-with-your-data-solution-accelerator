# BUG-0055 — MACAE Application Insights credential pattern research

Status: **Complete**

## Research question

Does the Microsoft Multi-Agent Custom Automation Engine Solution Accelerator (MACAE)
authenticate Azure Monitor / Application Insights ingestion with a **managed-identity
credential** (Entra token via `DefaultAzureCredential` / `ManagedIdentityCredential`)
or with a **connection-string / instrumentation-key only** (local auth)?

Context: validating a fix for CWYD where App Insights has `disableLocalAuth: true`
and telemetry must present an Entra token.

Repo: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator (branch `main`)

---

## VERDICT (one sentence)

**MACAE uses connection-string-only (local-auth / instrumentation-key) telemetry ingestion — it does NOT pass a `credential=` to `configure_azure_monitor(...)`, does NOT grant `Monitoring Metrics Publisher`, and (critically) does NOT set `disableLocalAuth` on its Application Insights component (local auth stays ENABLED), which is exactly why no Entra credential is needed.**

MACAE is therefore **NOT** a valid precedent for the CWYD fix: CWYD's App Insights has `disableLocalAuth: true`, so CWYD must present an Entra token; MACAE never has to.

---

## (a) Where MACAE inits Azure Monitor (file + code)

**File:** `src/backend/app.py` (FastAPI backend entry point).

Import (top of file):
```python
from azure.monitor.opentelemetry import configure_azure_monitor
...
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
```

Initialization block (module-level, after `app = FastAPI(lifespan=lifespan)`):
```python
frontend_url = config.FRONTEND_SITE_NAME
# Configure Azure Monitor and instrument FastAPI for OpenTelemetry
# This enables automatic request tracing, dependency tracking, and proper operation_id
if config.APPLICATIONINSIGHTS_CONNECTION_STRING:
    # Configure Application Insights telemetry with live metrics
    configure_azure_monitor(
        connection_string=config.APPLICATIONINSIGHTS_CONNECTION_STRING,
        enable_live_metrics=True
    )

    # Instrument FastAPI app — exclude WebSocket URLs to reduce telemetry noise
    FastAPIInstrumentor.instrument_app(
        app,
        excluded_urls="socket,ws"
    )
    logging.info("Application Insights configured with live metrics and WebSocket filtering")
else:
    logging.warning(
        "No Application Insights connection string found. Telemetry disabled."
    )
```

- Permalink (raw): https://raw.githubusercontent.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/main/src/backend/app.py
- Blob: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/app.py

There is a second, unrelated tracing helper `src/backend/common/utils/otlp_tracing.py` (`configure_oltp_tracing`) that builds a generic OpenTelemetry `TracerProvider` with an **`OTLPSpanExporter`** (gRPC OTLP), NOT the Azure Monitor exporter, and takes no credential. It is not the Azure Monitor path and is not what wires App Insights.
- https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/common/utils/otlp_tracing.py

---

## (b) DEFINITIVE answer: credential-based or connection-string-only?

**Connection-string-only (local auth / instrumentation key).**

Exact call with its full argument list (nothing elided):
```python
configure_azure_monitor(
    connection_string=config.APPLICATIONINSIGHTS_CONNECTION_STRING,
    enable_live_metrics=True
)
```

- **No `credential=` argument is passed.** Only `connection_string` + `enable_live_metrics`.
- `config.APPLICATIONINSIGHTS_CONNECTION_STRING` is a **required** env var loaded in `AppConfig.__init__` via `self._get_required("APPLICATIONINSIGHTS_CONNECTION_STRING")`.
  - `src/backend/common/config/app_config.py` — https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/common/config/app_config.py
- Because no credential is presented, ingestion relies on the connection string's instrumentation key (local auth). This only works because App Insights local auth is enabled (see (d)).

---

## (c) Credential type + sync/async + client-id source

**N/A for telemetry.** MACAE does **not** create or pass any credential to Azure Monitor.

MACAE *does* have a credential factory in `AppConfig`, but it is used for **other** Azure SDK clients (AI Project, Cosmos, OpenAI token provider) — **never** for `configure_azure_monitor`:

```python
def get_azure_credential(self, client_id=None):
    if self.APP_ENV == "dev":
        return DefaultAzureCredential(exclude_environment_credential=True)  # CodeQL [SM05139]
    else:
        return ManagedIdentityCredential(client_id=client_id)

def get_azure_credential_async(self, client_id=None):
    if self.APP_ENV == "dev":
        return DefaultAzureCredentialAsync(exclude_environment_credential=True)
    else:
        return ManagedIdentityCredentialAsync(client_id=client_id)
```
- Both sync and async variants exist. Client id comes from `self.AZURE_CLIENT_ID` (env `AZURE_CLIENT_ID`, wired in Bicep to the user-assigned identity's `clientId`).
- Consumed for the AzureOpenAI token provider in `src/backend/v4/config/settings.py` (`AzureConfig.ad_token_provider` → `self.credential.get_token(config.AZURE_COGNITIVE_SERVICES)`), i.e. for **model auth**, not telemetry.
  - https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/v4/config/settings.py
- File: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/common/config/app_config.py

**This credential is never handed to `configure_azure_monitor`.** So MACAE gives no answer to "which credential for telemetry" because it presents none.

---

## (d) MACAE Bicep: `disableLocalAuth` + `Monitoring Metrics Publisher`

**App Insights component — does NOT set `disableLocalAuth` (local auth stays ENABLED).**

The Application Insights component is deployed via AVM `avm/res/insights/component:0.7.1` with these params only:
```bicep
var applicationInsightsResourceName = 'appi-${solutionSuffix}'
module applicationInsights 'br/public:avm/res/insights/component:0.7.1' = if (enableMonitoring) {
  name: take('avm.res.insights.component.${applicationInsightsResourceName}', 64)
  params: {
    name: applicationInsightsResourceName
    tags: tags
    location: location
    enableTelemetry: enableTelemetry
    retentionInDays: 365
    kind: 'web'
    disableIpMasking: false
    flowType: 'Bluefield'
    workspaceResourceId: enableMonitoring ? logAnalyticsWorkspaceResourceId : ''
  }
}
```
- `disableLocalAuth` is **absent** → AVM default `false` → **local auth ENABLED** on App Insights. Instrumentation-key/connection-string ingestion is allowed. This is the root reason MACAE needs no credential.
- `infra/main.bicep` (~L361–376): https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main.bicep#L361-L376
- `infra/main_custom.bicep` (~L359–375): https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main_custom.bicep#L359-L375

**The only `disableLocalAuth: true` occurrences in MACAE Bicep are on OTHER resources, NOT App Insights:**
- `kind: 'AIServices'` → the AI Foundry / Cognitive Services account.
- `hostingMode: 'Default'` → the Azure AI Search service.
- (Present in both `infra/main.bicep` and `infra/main_custom.bicep`.)
- Source: `github_text_search` for `disableLocalAuth` returned exactly these two contexts per file — neither is the App Insights module.

**`Monitoring Metrics Publisher` role assignment — NONE.**
- `github_text_search` for `Monitoring Metrics Publisher` across the entire repo returned **empty**. MACAE grants no managed identity the telemetry-ingestion role, consistent with connection-string-only ingestion.

**How the connection string reaches the app:** Bicep injects `APPLICATIONINSIGHTS_CONNECTION_STRING` (and `APPLICATIONINSIGHTS_INSTRUMENTATION_KEY`) as plain container-app env vars:
```bicep
{ name: 'APPLICATIONINSIGHTS_CONNECTION_STRING', value: enableMonitoring ? applicationInsights!.outputs.connectionString : '' }
```
- `infra/main.bicep` (~L1287–1291): https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main.bicep#L1287-L1291

---

## (e) Agent-framework tracing path

- MACAE uses the **Microsoft Agent Framework** (`agent_framework`, `AzureOpenAIChatClient`) — see `src/backend/v4/config/settings.py`.
- **No separate agent-framework telemetry wiring configures a credential for Azure Monitor.** There is no `AIAgentsInstrumentor` / `setup_telemetry` / `configure_tracing`-with-credential path. Telemetry is wired purely via the single `configure_azure_monitor(connection_string=...)` call in `app.py` plus `FastAPIInstrumentor.instrument_app(app, excluded_urls="socket,ws")`.
- The agent-framework credential (`AzureConfig.credential` = `config.get_azure_credentials()`) is used only to mint an AD token for the OpenAI chat client scope (`AZURE_COGNITIVE_SERVICES`), not for tracing export.
- `src/backend/v4/config/settings.py`: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/v4/config/settings.py

---

## (f) Env gating

- Telemetry is gated **solely on the presence of the connection string**:
  ```python
  if config.APPLICATIONINSIGHTS_CONNECTION_STRING:
      configure_azure_monitor(connection_string=..., enable_live_metrics=True)
      FastAPIInstrumentor.instrument_app(app, excluded_urls="socket,ws")
  else:
      logging.warning("No Application Insights connection string found. Telemetry disabled.")
  ```
- No production-only gate. (`APP_ENV` — `"dev"` vs else — only switches which **credential class** is used for *non-telemetry* Azure SDK clients: `DefaultAzureCredential(exclude_environment_credential=True)` in dev vs `ManagedIdentityCredential(client_id=...)` otherwise. Bicep sets `APP_ENV: 'Prod'`.)
- Live metrics always on (`enable_live_metrics=True`); WebSocket URLs (`socket`, `ws`) excluded from instrumentation to cut noise.

---

## (g) Pinned versions

From `src/backend/pyproject.toml` and `src/backend/requirements.txt`:
- `azure-monitor-opentelemetry==1.8.7`
- `azure-identity==1.25.3`
- `azure-monitor-events-extension==0.1.0` (used by `common/utils/event_utils.py` → `track_event`)
- (context) `azure-cosmos==4.15.0`, `azure-search-documents==11.6.0`, `openai==2.33.0`, `fastapi==0.136.1`

- pyproject: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/pyproject.toml
- requirements: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/requirements.txt

---

## Implication for CWYD (BUG-0055)

- MACAE is **connection-string-only** and its App Insights **leaves local auth enabled**, so it is **not** a precedent for the credential-based fix CWYD needs.
- Because CWYD's App Insights sets `disableLocalAuth: true`, ingestion with only a connection string will fail — CWYD must pass a `credential=` to `configure_azure_monitor(...)`. The MACAE code cannot be copied for this; the CWYD fix must diverge from MACAE by (a) passing an Entra credential to `configure_azure_monitor`, and (b) granting `Monitoring Metrics Publisher` to the app's managed identity on the App Insights component in Bicep.
- The generic `azure-monitor-opentelemetry` API does accept a `credential=` kwarg (the SDK's documented pattern for AAD-auth ingestion); MACAE simply never uses it. Confirm the exact `credential=` signature against `azure-monitor-opentelemetry` (CWYD's pinned version) when implementing.

---

## References (file paths + URLs)

- `src/backend/app.py` — Azure Monitor init (connection-string-only). https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/app.py
- `src/backend/common/config/app_config.py` — credential factory (not used for telemetry). https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/common/config/app_config.py
- `src/backend/v4/config/settings.py` — agent-framework `AzureConfig` (credential for model auth only). https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/v4/config/settings.py
- `src/backend/common/utils/otlp_tracing.py` — unrelated OTLP gRPC exporter helper. https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/common/utils/otlp_tracing.py
- `src/backend/common/utils/event_utils.py` — `track_event_if_configured` (gated on connection string). https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/common/utils/event_utils.py
- `infra/main.bicep` — App Insights AVM module (no `disableLocalAuth`); App Insights env-var injection; `disableLocalAuth: true` only on AIServices + Search. https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main.bicep
- `infra/main_custom.bicep` — same pattern. https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main_custom.bicep
- `src/backend/pyproject.toml` / `src/backend/requirements.txt` — pinned versions.
- Search evidence: repo-wide `Monitoring Metrics Publisher` search → **empty** (no such role assignment).
