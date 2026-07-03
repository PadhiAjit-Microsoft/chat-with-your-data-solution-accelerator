# BUG-0055 — External Best-Practices Research: Azure App Insights + OpenTelemetry (Python) Zero-Telemetry

> Status: Complete
> Scope: READ-ONLY research reference. Diagnose why a Python FastAPI Container App + Azure Functions
> (Python) Container App send ZERO telemetry via `azure-monitor-opentelemetry` (`configure_azure_monitor()`).

## Research questions

1. How `configure_azure_monitor()` reads the connection string; default env var; behavior when unset/empty.
2. Ranked common causes of ZERO telemetry from a Python app.
3. Azure Functions (Python, Container App) telemetry; `host.json` sampling; host vs. worker OTel.
4. Entra-ID / managed-identity ingestion: `DisableLocalAuth` → 403; `credential=` requirement; role name.
5. Verification KQL queries + ingestion latency.
6. Environment-gating anti-pattern (only init telemetry on `environment == production`).

## Executive summary (TL;DR)

- **Default env var:** `configure_azure_monitor()` auto-populates its connection string from **`APPLICATIONINSIGHTS_CONNECTION_STRING`** when `connection_string=` is not passed explicitly. Explicit args always win over env vars.
- **When unset/empty:** The Learn "enable"/"configuration" pages do NOT explicitly document the unset behavior. Authoritative SDK behavior: the exporter's connection-string parser **raises `ValueError` ("Connection string is not set.")**, so `configure_azure_monitor()` fails fast at startup rather than silently no-op'ing — UNLESS the app wraps the call in a try/except (or an `environment`-gate) that swallows it, in which case you get a silent zero-telemetry app. This is the #1 real-world cause.
- **Top 3 zero-telemetry causes:** (1) connection string env var missing/empty (or the call is gated/try-excepted so it never runs); (2) `configure_azure_monitor()` never actually invoked on that process (import-order bug for FastAPI/Flask instrumentation, or gated behind `environment == production`); (3) `DisableLocalAuth=true` on the App Insights component while the app sends connection-string-only (no `credential=`) → ingestion rejected (401/403), OR sampling/exporter fully disabled (`OTEL_TRACES_SAMPLER=always_off`, `sampling_ratio=0`, `OTEL_*_EXPORTER=None`).
- **Entra ingestion requirements:** role = **Monitoring Metrics Publisher** (scoped to the App Insights resource — note it publishes ALL telemetry, not just metrics), AND the app must pass `credential=` (e.g. `ManagedIdentityCredential(client_id=...)` / `DefaultAzureCredential()`) to `configure_azure_monitor(credential=...)`. Token audience (public cloud) = `https://monitor.azure.com`.
- **Verify:** ingestion delay is "a few minutes" (typically ~1–3 min). Copy-paste KQL below.

---

## (a) How `configure_azure_monitor()` reads the connection string + behavior when unset

**Default env var — `APPLICATIONINSIGHTS_CONNECTION_STRING`.** Both the distro README and the Learn config page confirm this is the single env var read:

- Distro README (Usage table): *"connection_string — The connection string for your Application Insights resource. The connection string will be **automatically populated from the `APPLICATIONINSIGHTS_CONNECTION_STRING` environment variable** if not explicitly passed in."* And: *"All pass-in parameters take priority over any related environment variables."*
  - Source: https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry/README.md
- Learn (Configuration → Connection string, Python tab): two supported ways — set env var `APPLICATIONINSIGHTS_CONNECTION_STRING=<...>`, or pass `configure_azure_monitor(connection_string="<...>")`. Precedence (highest→lowest): **1. Code → 2. Environment variable → 3. Config file**.
  - Source: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration?tabs=python
- Learn (Enable): *"We recommend setting the connection string through code only in local development and test environments. For production, use an environment variable."*
  - Source: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable?tabs=python

**Behavior when the env var is unset / empty.** The Learn enable/configuration pages are **silent** on this exact case. The underlying exporter README confirms auto-population from the env var but does not state the empty case either:

- *"You can also instantiate the exporter directly via the constructor. In this case, the connection string will be automatically populated from the `APPLICATIONINSIGHTS_CONNECTION_STRING` environment variable."*
  - Source: https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry-exporter/README.md

  Authoritative runtime behavior (from the `azure-monitor-opentelemetry-exporter` connection-string parser, `_ConnectionStringParser`): when neither a `connection_string=` argument nor the `APPLICATIONINSIGHTS_CONNECTION_STRING` env var resolves to a value, the parser **raises `ValueError("Connection string is not set.")`**. Because the distro builds its exporters during `configure_azure_monitor()`, that call **raises at startup** — it is NOT a silent no-op. Practical consequence: a truly missing connection string normally crashes app boot loudly. A *silent* zero-telemetry app therefore almost always means the `configure_azure_monitor()` call is either (i) never reached, or (ii) wrapped in a `try/except`/environment gate that swallows the `ValueError`. Flag this as SDK-source-derived (not doc-quoted) — verify against the installed exporter version if the fix hinges on it.

**Key `configure_azure_monitor()` parameters relevant to this bug** (distro README):

| Param | Meaning / default |
| --- | --- |
| `connection_string` | Falls back to `APPLICATIONINSIGHTS_CONNECTION_STRING`. |
| `credential` | Token credential for Entra ingestion (e.g. `ManagedIdentityCredential`). Default `None`. Auto-created from `APPLICATIONINSIGHTS_AUTHENTICATION_STRING` if not passed. |
| `logger_name` | Namespace of the Python logger whose logs are collected. **Setting this is important** so SDK-internal logs aren't tracked; if your app logs on a logger that is NOT this logger or a child of it, those logs won't be exported. |
| `sampling_ratio` | Application Insights fixed-% sampler ratio, 0.0–1.0. **Default 1.0 (100%).** `0` = drop all traces. |
| `traces_per_second` | Enables the Rate-Limited sampler at N traces/sec. |
| `enable_live_metrics` / `enable_performance_counters` | Default `True`. |

- Source: https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry/README.md

**Default sampler.** If you set neither the sampling env vars nor `sampling_ratio`/`traces_per_second`, `configure_azure_monitor()` uses the **Rate-Limited sampler at a default of 5.0 traces/sec** (distro README OTEL_TRACES_SAMPLER row + Learn config note: *"If you don't set any environment variables or provide either `sampling_ratio` or `traces_per_second`, `configure_azure_monitor()` uses RateLimitedSampler by default."*). Note `sampling_ratio` itself defaults to 1.0 (100%). So default config does NOT drop telemetry.
  - Sources: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration?tabs=python ; https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry/README.md

**Flush on exit.** *"telemetry is flushed automatically upon application exit. Note that this does not include when application ends abruptly or crashes due to uncaught exception."* Default export intervals: traces `OTEL_BSP_SCHEDULE_DELAY`=5000 ms, logs `OTEL_BLRP_SCHEDULE_DELAY`=5000 ms, metrics `OTEL_METRIC_EXPORT_INTERVAL`=60000 ms. Failed telemetry is cached to offline storage and retried for up to **48 hours**. So a short-lived process that exits before the interval can still lose telemetry if it crashes on an uncaught exception, but a long-running FastAPI/Container App process does not have this problem.
  - Sources: https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry-exporter/README.md ; https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration?tabs=python

---

## (b) Ranked list of common zero-telemetry causes

Ranked most→least likely for a long-running FastAPI Container App + Functions Container App:

1. **Connection string env var missing/empty on the running container.** `APPLICATIONINSIGHTS_CONNECTION_STRING` not injected into the Container App (or blank). If `configure_azure_monitor()` is reached, it raises `ValueError` at boot; if the call is gated or try/excepted, you get silent zero telemetry. **Verify in-container:** `echo $APPLICATIONINSIGHTS_CONNECTION_STRING`. (Learn config connection-string section.)
2. **`configure_azure_monitor()` is never actually executed on that process** — the module holding the call isn't imported at startup, or it's behind an `if environment == "production":` gate that is false, or wrapped in `try/except` that swallows the error. (Anti-pattern; see section (f).)
3. **`DisableLocalAuth=true` on the App Insights component but the app uses connection-string-only auth (no `credential=`).** Ingestion is rejected because only Entra-authenticated telemetry is accepted. Manifests as auth failures (401/403) in exporter diagnostic logs. (See section (d).)
4. **FastAPI/Flask request telemetry missing due to import order** — importing `fastapi.FastAPI`/`flask.Flask` (the class) BEFORE calling `configure_azure_monitor()` means the instrumentation never patches the framework. Symptom: `requests`/`AppRequests` table empty but other tables may have data. Fix: import the module (`import fastapi`) and call `configure_azure_monitor()` before referencing `fastapi.FastAPI`, or call configure before importing the class.
   - Source: https://learn.microsoft.com/en-us/troubleshoot/azure/azure-monitor/app-insights/telemetry/opentelemetry-troubleshooting-python
5. **Sampling / exporter fully disabled.** `OTEL_TRACES_SAMPLER=always_off` (or `microsoft.fixed_percentage` / `trace_id_ratio` with arg `0.0`), or `sampling_ratio=0`. Note logs belonging to unsampled traces are dropped by default; metrics are never sampled. Or a signal exporter turned off entirely: `OTEL_TRACES_EXPORTER=None` / `OTEL_LOGS_EXPORTER=None` / `OTEL_METRICS_EXPORTER=None`. Env vars take precedence over code.
   - Sources: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration?tabs=python ; distro README.
6. **Network egress blocked / AMPLS / firewall** — the container cannot reach the ingestion endpoint in the connection string. Test with cURL/REST from inside the host to the `IngestionEndpoint`. (Learn Python troubleshooting → "Test connectivity between your application host and the ingestion service".)
   - Source: https://learn.microsoft.com/en-us/troubleshoot/azure/azure-monitor/app-insights/telemetry/opentelemetry-troubleshooting-python
7. **Wrong-region / wrong-resource connection string** — telemetry lands in a different App Insights component than the one being queried. The `IngestionEndpoint` in the connection string dictates the region/resource; querying the wrong resource shows "nothing arrived."
8. **Logging not attached to the OTel handler** — the app logs on a logger that isn't the `logger_name` passed to `configure_azure_monitor()` (nor a child of it), so log/trace telemetry from `logging` calls is never captured. Fix: set `logger_name` to your app's namespace and log through it (or a child).
   - Source: https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry/README.md ; exporter README ("use a named logger rather than the root logger").
9. **Process crashes before flush** — short-lived process exits via uncaught exception before the 5 s export interval; telemetry not flushed. Less likely for a long-running server. Call `force_flush()` / provider `shutdown()` in short-lived paths.
   - Source: exporter README (Flush/shutdown behavior).

**Diagnostic first move:** enable SDK diagnostic logging to see export attempts/failures directly:
```python
import logging
logging.basicConfig(format="%(asctime)s:%(levelname)s:%(message)s", level=logging.DEBUG)
```
Source: https://learn.microsoft.com/en-us/troubleshoot/azure/azure-monitor/app-insights/telemetry/opentelemetry-troubleshooting-python

---

## (c) Azure Functions-specific telemetry + `host.json` sampling

**Two distinct telemetry paths** (do not conflate):

1. **Built-in Application Insights integration (classic, default).** The Functions **host** sends telemetry using the App Insights SDK automatically when `APPLICATIONINSIGHTS_CONNECTION_STRING` is present — requests, dependencies, exceptions, host traces — with **no code**. This is what `host.json` `logging.applicationInsights.samplingSettings` and `logging.logLevel` govern.
2. **OpenTelemetry with Azure Monitor Exporter (recommended).** Enabled by `host.json` `"telemetryMode": "OpenTelemetry"` + `APPLICATIONINSIGHTS_CONNECTION_STRING`. **The Azure Monitor OpenTelemetry Exporter requires the connection string and does NOT support an instrumentation key.**
   - Source: https://learn.microsoft.com/en-us/azure/azure-functions/functions-monitoring ("The Azure Monitor OpenTelemetry Exporter requires an Application Insights connection string (`APPLICATIONINSIGHTS_CONNECTION_STRING`) and doesn't support the use of an instrumentation key.")

**Does the Functions app need `APPLICATIONINSIGHTS_CONNECTION_STRING`? Yes.** For both the built-in integration and OTel-to-App-Insights, the connection string app setting is required (instrumentation key is legacy/deprecated; sovereign clouds require the connection string).
  - Source: https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring (Enable Application Insights integration).

**Python worker telemetry — host vs worker.** The Functions **host** auto-instruments and emits telemetry; the **Python worker** does NOT automatically export rich OTel telemetry from your function code unless you either:
- set app setting **`PYTHON_APPLICATIONINSIGHTS_ENABLE_TELEMETRY=true`** (lets the host allow the Python worker to stream OpenTelemetry logs directly, preventing duplicate host-level entries), OR
- call **`configure_azure_monitor()`** in `function_app.py` yourself (manual enablement).
  - Source: https://learn.microsoft.com/en-us/azure/azure-functions/opentelemetry-howto (Python tab).

  Note: the Functions worker ALSO emits logging telemetry itself without the SDK → **duplicate log entries** if you also call `configure_azure_monitor()`. Mitigations: clear root-logger handlers before configuring, or set `OTEL_LOGS_EXPORTER=None` to keep only the native Functions logging.
  - Sources: distro README (Logging issues) ; https://learn.microsoft.com/en-us/troubleshoot/azure/azure-monitor/app-insights/telemetry/opentelemetry-troubleshooting-python (Duplicate trace logs in Azure Functions).

**`host.json` sampling — `logging.applicationInsights.samplingSettings`** (v2.x+ schema):
```json
{
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 20,
        "excludedTypes": "Request;Exception"
      }
    }
  }
}
```
- **`isEnabled` default: `true`.** Adaptive sampling is ON by default. Default `maxTelemetryItemsPerSecond` = **20** (5 in Functions v1.x).
- **Can sampling drop everything? Not by itself.** Adaptive sampling randomly drops *excess* items only when the incoming rate exceeds the threshold; it statistically preserves a representative sample and adjusts item counts. It won't zero out a low-traffic app. Use `excludedTypes` (e.g. `"Request;Exception"`) to guarantee all requests/exceptions are always kept. Sampling is applied *per host instance*.
  - Sources: https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring (Configure sampling) ; https://learn.microsoft.com/en-us/azure/azure-functions/functions-monitoring (sampling enabled by default).

**Much more likely Functions "zero telemetry" cause than sampling — `host.json` log levels.** `logging.logLevel` settings apply globally to the .NET logging pipeline and gate what reaches App Insights tables. Setting a category to anything other than `Information` (e.g. `None`, `Warning`, `Error`) **prevents telemetry from flowing to the corresponding table**:
- `Host.Results` → `requests` table (function success/failure). If set above `Information`, successful executions vanish from `requests`.
- `Host.Aggregator` → `customMetrics` (invocation counts). Above `Information` → Overview metrics vanish.
- `Function` → `traces`/`dependencies`/`customMetrics`/`customEvents`. Set to `Error` → only error traces collected; dependencies/custom telemetry dropped.
- Caution (verbatim): *"If you set a category log level to any value different from `Information`, it prevents the telemetry from flowing to those tables, and you won't be able to see related data."* A `default: "None"` (or too-high default) silences everything.
  - Source: https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring (Configure log levels).

**Critical OTel-mode gotcha:** *"If you set `telemetryMode` to `OpenTelemetry`, the configuration in the `logging.applicationInsights` section of host.json doesn't apply."* So `samplingSettings` is a no-op under OTel mode — in OTel mode, control sampling via the standard OTel env vars instead. Also: filters in `host.json` apply only to **host-process** logs; **worker-process** logs are filtered via language-specific OpenTelemetry settings, not host.json.
  - Source: https://learn.microsoft.com/en-us/azure/azure-functions/opentelemetry-howto (Considerations; Log filtering).

**OTel parent-based sampling gotcha (missing request telemetry):** with parent-based sampling as default, request telemetry for triggers (HTTP, Service Bus, Event Hubs) isn't generated when the incoming request/message isn't sampled.
  - Source: https://learn.microsoft.com/en-us/azure/azure-functions/opentelemetry-howto (Missing request telemetry).

---

## (d) Entra-ID / managed-identity ingestion requirements

When **`DisableLocalAuth: true`** is set on the App Insights component (`Microsoft.Insights/components` property), only Microsoft Entra-authenticated telemetry is ingested; connection-string-only ("local auth") ingestion is rejected. To keep telemetry flowing you MUST:

1. **Assign the `Monitoring Metrics Publisher` role** to the app's identity (managed identity / service principal / user), scoped to the target **Application Insights resource**.
   - Verbatim: *"Follow the steps in Assign Azure roles to add the **Monitoring Metrics Publisher** role to the expected identity... by setting the target Application Insights resource as the role scope."*
   - Verbatim note: *"Although the Monitoring Metrics Publisher role says 'metrics,' it publishes ALL telemetry to the Application Insights resource."*
   - Source: https://learn.microsoft.com/en-us/azure/azure-monitor/app/azure-ad-authentication?tabs=python
2. **Pass a `credential=` to `configure_azure_monitor(...)`.** Python examples (verbatim from the Entra auth page, Python tab):
   ```python
   from azure.identity import ManagedIdentityCredential
   from azure.monitor.opentelemetry import configure_azure_monitor

   credential = ManagedIdentityCredential(client_id="<client_id>")  # user-assigned MI
   configure_azure_monitor(
       connection_string="your-connection-string",
       credential=credential,
   )
   ```
   - System-assigned MI: `ManagedIdentityCredential()` (no args). Local dev: `DefaultAzureCredential()`. Service principal: `ClientSecretCredential(...)`.
   - The connection string is STILL required (it supplies the ingestion endpoint); the credential supplies the auth token.
   - Alternatively the exporter auto-creates the credential from the `APPLICATIONINSIGHTS_AUTHENTICATION_STRING` env var (`Authorization=AAD` for SAMI, `Authorization=AAD;ClientId=<...>` for UAMI). Functions can use `APPLICATIONINSIGHTS_AUTHENTICATION_STRING` at the host level.
   - Sources: https://learn.microsoft.com/en-us/azure/azure-monitor/app/azure-ad-authentication?tabs=python ; https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry-exporter/README.md ; https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring (Require Microsoft Entra authentication — confirms the identity needs a role "equivalent to Monitoring Metrics Publisher" and that `APPLICATIONINSIGHTS_CONNECTION_STRING` is still required).
3. **Token audience** (custom clients): public cloud `https://monitor.azure.com`; 21Vianet `https://monitor.azure.cn`; US Gov `https://monitor.azure.us`.
   - Source: https://learn.microsoft.com/en-us/azure/azure-monitor/app/azure-ad-authentication?tabs=python

**Functions + OTel + Entra:** you must configure Entra auth **separately for both the host process and the worker process** (host via `configure-monitoring#require-microsoft-entra-authentication`; worker via the azure-ad-authentication page).
  - Source: https://learn.microsoft.com/en-us/azure/azure-functions/opentelemetry-howto (Microsoft Entra authentication).

**Symptom when misconfigured:** `DisableLocalAuth=true` + no credential (or identity lacks Monitoring Metrics Publisher) → ingestion is rejected (auth failure, 401/403) → **zero telemetry** despite a valid connection string. This exactly matches the bug profile. Troubleshooting pointer: "Troubleshoot Microsoft Entra authentication issues" in the investigate-missing-telemetry guide.

---

## (e) Verification KQL + ingestion latency

**Ingestion latency:** Learn (Enable → "Confirm data is flowing"): *"It might take a few minutes for data to show up."* Typical end-to-end ingestion delay is ~**1–3 minutes** (can be longer under load or when offline-storage retry is engaged). Wait at least a few minutes before concluding "nothing arrived."
  - Source: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable?tabs=python

**Schema note:** the App Insights resource **Logs** blade uses the *classic* table names (`requests`, `traces`, `dependencies`, `exceptions`, `customMetrics`, `customEvents`, `pageViews`, `availabilityResults`). A workspace-based App Insights queried through the **Log Analytics workspace** uses the `App*` table names (`AppRequests`, `AppTraces`, `AppDependencies`, `AppExceptions`, `AppMetrics`, `AppEvents`). Use whichever matches where you run the query.

**KQL 1 — Did ANY telemetry arrive in the last hour? (classic / AI Logs blade):**
```kusto
union isfuzzy=true requests, traces, dependencies, exceptions, customMetrics, customEvents, pageViews, availabilityResults
| where timestamp > ago(1h)
| summarize items = count() by itemType
| order by items desc
```

**KQL 2 — Last-received timestamp per telemetry type (classic):**
```kusto
union isfuzzy=true requests, traces, dependencies, exceptions, customMetrics
| summarize LastSeen = max(timestamp), items = count() by itemType
| order by LastSeen desc
```

**KQL 3 — Which app/role is emitting (classic, last 24h) — confirms cloud role name mapping:**
```kusto
union isfuzzy=true requests, traces, dependencies, exceptions
| where timestamp > ago(24h)
| summarize LastSeen = max(timestamp), items = count() by cloud_RoleName, itemType
| order by LastSeen desc
```

**KQL 4 — Absolute smallest "anything at all?" probe (classic):**
```kusto
union isfuzzy=true requests, traces, dependencies, exceptions, customMetrics, customEvents, pageViews, availabilityResults, browserTimings
| where timestamp > ago(24h)
| count
```

**KQL 5 — Workspace-based (Log Analytics `App*` tables), last hour:**
```kusto
union isfuzzy=true AppRequests, AppTraces, AppDependencies, AppExceptions, AppMetrics
| where TimeGenerated > ago(1h)
| summarize items = count() by Type
| order by items desc
```

**KQL 6 — Workspace-based, last-seen per cloud role (find the right app / confirm arrival):**
```kusto
union isfuzzy=true AppRequests, AppTraces, AppDependencies, AppExceptions
| where TimeGenerated > ago(24h)
| summarize LastSeen = max(TimeGenerated), items = count() by AppRoleName, Type
| order by LastSeen desc
```

If KQL 1/4/5 return `0`, telemetry is genuinely not being ingested (env var / never-configured / auth / egress). If requests are absent but traces/dependencies present, suspect the FastAPI/Flask import-order bug (section (b) #4) or Functions `Host.Results` log-level gating (section (c)).

---

## (f) Environment-gating anti-pattern

**Confirmed recommended pattern: always configure telemetry when a connection string is present, regardless of `environment`.**

- The Learn guidance is unconditional: set the connection string via env var in production and call `configure_azure_monitor()` — there is no documented "only enable in production" gate. The only documented environment distinction is *where you put the connection string* (code for local dev/test; env var for production), not *whether* to configure telemetry.
  - Source: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable?tabs=python
- Anti-pattern in app code: `if settings.environment == "production": configure_azure_monitor()`. If the deployed Container App's `environment`/mode value is not exactly `"production"` (e.g. it defaults to `local` because the prod env var was never wired by IaC), telemetry is silently disabled even though a valid `APPLICATIONINSIGHTS_CONNECTION_STRING` is present. This is a very common real cause of "zero telemetry in cloud."
- **Recommended gate instead:** key off *presence of the connection string*, not the environment:
  ```python
  import os
  from azure.monitor.opentelemetry import configure_azure_monitor

  conn = os.environ.get("APPLICATIONINSIGHTS_CONNECTION_STRING")
  if conn:  # configure whenever a connection string exists, in any environment
      configure_azure_monitor(logger_name="myapp")
  ```
  This lets local dev run telemetry-free (no connection string) while every deployed runtime that has the env var exports telemetry — no environment coupling. (Aligns with the repo memory note "config defaults dev/local-first, prod via env vars": the prod env var, not an app-code `environment` check, flips the runtime on.)

**Cross-check for this bug:** if the app currently gates on `environment == "production"` AND the Container Apps (backend + function app) never received the prod `environment`/`AZURE_ENVIRONMENT` env var from IaC, the gate is false and `configure_azure_monitor()` never runs → zero telemetry with a perfectly valid connection string. Fix by (a) wiring the prod env var in Bicep on every deployed runtime, and/or (b) switching the gate to "connection string present."

---

## Clarifying questions

- None blocking. To convert this reference into a fix, the diagnosing agent should confirm on the running containers: (1) is `APPLICATIONINSIGHTS_CONNECTION_STRING` actually set and non-empty? (2) is `configure_azure_monitor()` gated behind an `environment`/`if` check or wrapped in `try/except`? (3) is `DisableLocalAuth=true` on the App Insights component, and if so is a `credential=` passed AND does the identity hold Monitoring Metrics Publisher on that resource? (4) for Functions: `telemetryMode`, `host.json` `logging.logLevel` defaults, and `PYTHON_APPLICATIONINSIGHTS_ENABLE_TELEMETRY`.

---

## Sources

- Enable OpenTelemetry in Application Insights (Python): https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable?tabs=python
- Configuring OpenTelemetry in Application Insights (Python — connection string, sampling, env vars, metric export interval, offline storage): https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration?tabs=python
- azure-monitor-opentelemetry distro (PyPI): https://pypi.org/project/azure-monitor-opentelemetry/
- azure-monitor-opentelemetry distro README (Usage params, sampler defaults, Functions logging issues): https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry/README.md
- azure-monitor-opentelemetry-exporter README (connection-string auto-population, credential, flush/shutdown, named-logger guidance): https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/monitor/azure-monitor-opentelemetry-exporter/README.md
- Troubleshoot OpenTelemetry issues in Python (diagnostic logging, connectivity test, FastAPI/Flask import order, duplicate telemetry): https://learn.microsoft.com/en-us/troubleshoot/azure/azure-monitor/app-insights/telemetry/opentelemetry-troubleshooting-python
- Monitor executions in Azure Functions (exporter requires connection string, sampling default on): https://learn.microsoft.com/en-us/azure/azure-functions/functions-monitoring
- Use OpenTelemetry with Azure Functions (telemetryMode, PYTHON_APPLICATIONINSIGHTS_ENABLE_TELEMETRY, host vs worker, samplingSettings-doesn't-apply-in-OTel-mode, parent-based sampling): https://learn.microsoft.com/en-us/azure/azure-functions/opentelemetry-howto
- Configure monitoring for Azure Functions (host.json samplingSettings schema, log-level gating of tables, Require Microsoft Entra authentication → Monitoring Metrics Publisher): https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring
- Microsoft Entra authentication for Application Insights (Monitoring Metrics Publisher role, DisableLocalAuth, credential= Python examples, token audience): https://learn.microsoft.com/en-us/azure/azure-monitor/app/azure-ad-authentication?tabs=python
