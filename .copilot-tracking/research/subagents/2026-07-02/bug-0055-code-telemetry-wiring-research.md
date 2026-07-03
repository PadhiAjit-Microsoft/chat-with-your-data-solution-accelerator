# BUG-0055 — Application-code telemetry wiring research

Status: **Complete** (read-only investigation)
Date: 2026-07-02
Scope: Find every place in the CWYD v2 **application code (Python)** where
OpenTelemetry / Azure Monitor telemetry is configured, and identify why no
telemetry would be exported to `appi-<SUFFIX>` from the backend Container App
and the Functions Container App.

> All Azure identifiers below use placeholders (`<SUFFIX>`, `<DATA_SUFFIX>`,
> `<RESOURCE_GROUP>`). Real values live only in the gitignored
> `v2/.azure/<AZD_ENV_NAME>/.env`.

---

## Research questions

1. Summarize the current BUG-0055 detail (both halves, prior fix attempts).
2. Where does the FastAPI backend init OpenTelemetry / Azure Monitor?
3. What env var / settings field carries the App Insights connection string?
4. Is telemetry init gated on a condition (environment / empty conn string)?
5. How do the Azure Functions configure telemetry? (`function_app.py`, `host.json`)
6. `host.json` sampling — could it drop everything?
7. Dependencies + versions.
8. Is telemetry gated on `environment == production` / `AZURE_ENVIRONMENT`?
   (cross-check BUG-0069).

---

## (a) BUG-0055 current-state summary

Source: `v2/docs/bugs.md` lines 998–1010 (`### BUG-0055 — Application Insights
receives zero telemetry from the function + backend`). Area: infra. Severity:
medium. **Status: open** (found 2026-06-16).

- **Symptom.** `appi-<SUFFIX>` has *never* received telemetry. A union query over
  `requests` / `traces` / `exceptions` / `dependencies` / `customMetrics`
  returned `[0, null, null]` (count 0, no min/max timestamp).
- **Not a connection-string mismatch.** The function's
  `APPLICATIONINSIGHTS_CONNECTION_STRING` was verified to match `appi-<SUFFIX>`
  exactly (same ingestion endpoint + instrumentation key).
- **Platform logs DO flow.** The host emits ~500 `Information` rows/24h to the
  `log-<SUFFIX>` Log Analytics workspace via its `allLogs` diagnostic setting
  (`FunctionAppLogs`). So the gap is specifically the **App Insights ingestion
  path** — the SDK/OTel *application* telemetry (`requests`/`traces`/`dependencies`)
  is not being exported, while the platform resource-log path (wired by the
  diagnostic setting, not the SDK) is unaffected.

### "Both halves code-complete" refers to two independent fixes landed 2026-06-23

- **Backend half — root cause CONFIRMED + fixed in Bicep (2026-06-23).** The
  backend ACA container is a plain container with **no host-level App Insights
  agent**, so its Python lifespan (`backend/app.py`) only emits telemetry when
  it calls `configure_azure_monitor(connection_string=...)`. That string comes
  from `ObservabilitySettings.app_insights_connection_string`, and
  `ObservabilitySettings` uses `env_prefix="AZURE_"` → it reads
  **`AZURE_APP_INSIGHTS_CONNECTION_STRING`** (a deliberate CU-002b/CU-007
  decision, enforced by two `test_app_lifespan.py` tests that intentionally
  ignore the standard name). But `main.bicep` had wired the backend container
  with the **standard** `APPLICATIONINSIGHTS_CONNECTION_STRING` (the name the
  Function host needs), so the typed setting stayed empty and
  `configure_azure_monitor` never fired in the cloud → zero backend telemetry.
  Local dev worked only because `v2/.env` uses the `AZURE_`-prefixed name.
  **Fix:** rename the *backend* container's env var to
  `AZURE_APP_INSIGHTS_CONNECTION_STRING` in Bicep (Function App keeps
  `APPLICATIONINSIGHTS_CONNECTION_STRING`). "Durable in Bicep — takes effect on
  the next `azd provision`; **not yet cloud-verified** (deploy blocked by the
  platform Flex build outage)." See ADR-0018 Amendment 1.
- **Function half — worker OTel wired (2026-06-23).** Added `configure_telemetry()`
  in `src/functions/core/telemetry.py`, called once at module load from
  `function_app.py`, which reads the host-provided
  `APPLICATIONINSIGHTS_CONNECTION_STRING` and calls
  `configure_azure_monitor(connection_string=...)` so the Python worker's
  application telemetry (logs/traces/dependencies) exports; absent it no-ops.
  Two `tests/functions/core/test_telemetry.py` tests cover configured + no-op.
  **Explicit caveat in the bug:** host-level `requests` (invocation) telemetry
  is emitted by the Functions **host**, not the worker — "if those stay absent
  after a deploy, enabling host OpenTelemetry (`host.json` `telemetryMode`) is a
  separate follow-up."
- **Bottom line in the bug:** "Both halves are now code/config-complete but **not
  yet cloud-verified** (deploy blocked by the Flex build outage); keep open
  until App Insights shows backend + function telemetry post-deploy."

### Material change AFTER the BUG-0055 text was last edited

BUG-0058 resolution (`v2/docs/bugs.md`, 2026-07-02) records that the **function
was migrated from Flex Consumption to a Container App built from
`docker/Dockerfile.functions`** (ACR remote build), and that a live
`azd deploy function` (3m16s) shipped current source on 2026-07-02. This
post-dates the BUG-0055 narrative, which still describes the function as a Flex
Consumption Function App "whose host reads `APPLICATIONINSIGHTS_CONNECTION_STRING`
natively." That assumption should be re-validated for the containerized host
(see §g, root cause candidates 2–3).

---

## (b) Backend telemetry wiring — exact files / lines / code

### Import — `v2/src/backend/app.py` line 25

```python
from azure.monitor.opentelemetry import configure_azure_monitor  # pyright: ignore[reportUnknownVariableType]
```

### Init — `v2/src/backend/app.py` lines 67–77 (inside `_lifespan`)

```python
@asynccontextmanager
async def _lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    settings = get_settings()

    conn_str = settings.observability.app_insights_connection_string.strip()
    if conn_str:
        configure_azure_monitor(connection_string=conn_str)
        logger.info("Application Insights telemetry configured.")
    else:
        logger.info(
            "AZURE_APP_INSIGHTS_CONNECTION_STRING not set; telemetry disabled."
        )
```

Module docstring (lines 6–10) states the design intent: telemetry is
configured to export **directly** to App Insights when
`ObservabilitySettings.app_insights_connection_string` is set; otherwise it is a
no-op so the backend-only profile boots with **no sidecar**.

Notes:

- No `FastAPIInstrumentor`, no `OpenTelemetryMiddleware`, no manual
  `set_tracer_provider`, no explicit span/exporter wiring in app code —
  `configure_azure_monitor()` (the azure-monitor-opentelemetry Distro) is the
  single entry point. It auto-instruments the installed OTel instrumentation
  libraries (see §f for the httpx instrumentation dep).
- The only additional instrumentation dependency declared is
  `opentelemetry-instrumentation-httpx` (§f).

### Related read-only reference — `v2/src/backend/routers/admin.py` line 132

```python
obs_conn = settings.observability.app_insights_connection_string.strip()
```

This is **not** a telemetry export path — it feeds the admin runtime-config
report (`app_insights_enabled = bool(conn)`; see `v2/docs/admin_runtime_config.md`
line 46). Mentioned only to disambiguate the grep hits.

---

## (c) Connection-string env var + settings field

### Backend (typed Pydantic settings)

`v2/src/backend/core/settings.py` lines 281–287:

```python
class ObservabilitySettings(BaseSettings):
    """OpenTelemetry / App Insights wiring (optional)."""

    model_config = SettingsConfigDict(env_prefix="AZURE_", extra="ignore")

    app_insights_connection_string: str = ""
    log_level: str = "INFO"
```

- Wired onto the root at `settings.py` line 551:
  `observability: ObservabilitySettings = Field(default_factory=ObservabilitySettings)`.
- **Env var name:** `env_prefix="AZURE_"` + field `app_insights_connection_string`
  → **`AZURE_APP_INSIGHTS_CONNECTION_STRING`**.
- **Default:** `""` (empty string).
- `.env.sample` (lines 69–76) documents this explicitly: "AppSettings reads the
  `AZURE_`-prefixed name (env_prefix=AZURE_); the legacy bare
  `APPLICATIONINSIGHTS_CONNECTION_STRING` is intentionally **not** honored
  post-CU-002b."

### Functions worker (raw env read, not typed settings)

`v2/src/functions/core/telemetry.py`:

```python
_APPLICATIONINSIGHTS_CONNECTION_STRING = "APPLICATIONINSIGHTS_CONNECTION_STRING"   # line 24

def configure_telemetry() -> bool:                                                # line 27
    conn_str = os.environ.get(_APPLICATIONINSIGHTS_CONNECTION_STRING, "").strip()  # line 36
    if not conn_str:                                                              # line 37
        logger.info("APPLICATIONINSIGHTS_CONNECTION_STRING not set; function telemetry disabled.")
        return False
    configure_azure_monitor(connection_string=conn_str)                          # line 44
    logger.info("Application Insights telemetry configured for functions.")
    return True
```

- **Env var name:** the **standard** `APPLICATIONINSIGHTS_CONNECTION_STRING`
  (read directly via `os.environ.get`, not through `ObservabilitySettings`).
- **Default:** effectively `""` (`os.environ.get(..., "")`).
- **Asymmetry is deliberate:** the two runtimes read *different* env-var names —
  backend `AZURE_APP_INSIGHTS_CONNECTION_STRING`, functions
  `APPLICATIONINSIGHTS_CONNECTION_STRING`. Documented in the telemetry module
  docstring (lines 6–11) and ADR-0018.

### Bicep wiring (what the deployed containers actually receive)

`v2/infra/main.bicep`:

- **Backend container** (lines 1904–1919), gated on `enableMonitoring`:
  ```bicep
  enableMonitoring
    ? [
        {
          name: 'AZURE_APP_INSIGHTS_CONNECTION_STRING'
          value: applicationInsights!.outputs.connectionString
        }
      ]
    : []
  ```
- **Functions container** (lines 2188–2197), gated on `enableMonitoring`:
  ```bicep
  enableMonitoring
    ? [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: applicationInsights!.outputs.connectionString
        }
      ]
    : []
  ```
- `param enableMonitoring bool = true` (line 208) — default ON.
- `module applicationInsights ... = if (enableMonitoring)` (line 316) — App
  Insights is only created when monitoring is on.

The Bicep now emits the **correct** name for each runtime. This matches the
"backend half fixed in Bicep" claim in BUG-0055. **The critical caveat: Bicep
env-var changes only apply on `azd provision` / `azd up`, NOT on `azd deploy`
(image-only push).**

---

## (d) Gating condition (the key suspect)

**Both** telemetry inits are gated **only** on the connection string being
non-empty. Neither is gated on `environment` / `AZURE_ENVIRONMENT` / a feature
flag.

- Backend: `if conn_str:` (`app.py` line 70) — empty ⇒ silently logs "telemetry
  disabled" and skips.
- Functions: `if not conn_str: ... return False` (`telemetry.py` line 37) —
  empty ⇒ no-op.

So the *only* code-level gate is: **is the connection-string env var present and
non-empty in the deployed container's environment?**

- Backend container ⇒ needs `AZURE_APP_INSIGHTS_CONNECTION_STRING` populated.
- Functions container ⇒ needs `APPLICATIONINSIGHTS_CONNECTION_STRING` populated.

If the deployed revision's env var uses the *wrong* name (or is absent because
that revision predates the Bicep fix / was image-pushed without a re-provision),
the string is empty and the exporter never initializes — exactly the confirmed
backend root cause in BUG-0055.

**Ruled out — `environment == production` gating.** There is NO
`if environment == production` / `is_production` guard anywhere around
telemetry. `AppSettings.environment` (`settings.py` line 533,
`Environment.LOCAL` default) is used for the local-dev identity bypass in chat,
**not** for telemetry. Therefore BUG-0069's unset-`AZURE_ENVIRONMENT` issue does
**not** suppress telemetry export. (See §h.)

---

## (e) Functions telemetry wiring + host.json sampling

### `function_app.py` — `v2/src/functions/function_app.py`

- Line 19: `from functions.core.telemetry import configure_telemetry`
- Lines 24–26 (module load, before function registration):
  ```python
  # Wire Azure Monitor export before registering functions (no-op when the
  # App Insights connection string is absent). Logic lives in
  # functions.core.telemetry so this module stays a thin registration surface.
  configure_telemetry()
  ```
- The rest of the module registers 5 ingestion blueprints + one anonymous
  `health` route. No other telemetry code.

So the worker **does** self-configure Azure Monitor at import time (matching the
"function half wired" claim). This is independent of any host-level integration.

### `host.json` — `v2/src/functions/host.json` (full file, lines 1–21)

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  },
  "extensions": {
    "queues": {
      "messageEncoding": "none"
    }
  }
}
```

Observations:

- `logging.applicationInsights.samplingSettings.isEnabled = true`,
  `excludedTypes = "Request"` — **adaptive sampling ON**, but requests are
  excluded from sampling (all requests kept). Adaptive sampling *thins*
  high-volume telemetry; it does **not** drop everything to absolute zero
  (`[0, null, null]`). So sampling is **not** the zero-telemetry cause.
- This `logging.applicationInsights` block configures the **host's** classic
  App Insights pipeline. It does **not** govern the Python worker's independent
  `configure_azure_monitor` OTel exporter — the worker export bypasses this
  block entirely.
- **No `logLevel` overrides** are present (no `logging.logLevel` section), so
  host logs default to `Information` and are not being filtered to nothing.
- **No `telemetryMode`** key — host-level OpenTelemetry is **not** enabled. For
  a classic (non-OTel) host, host-emitted `requests` telemetry depends on the
  host's built-in App Insights integration reading
  `APPLICATIONINSIGHTS_CONNECTION_STRING`. BUG-0055 already flags host `requests`
  as a **separate follow-up** even after the worker exports.

### `Dockerfile.functions` — `v2/docker/Dockerfile.functions`

- Installs deps by extracting `[project.dependencies]` from `v2/pyproject.toml`
  at build time (lines 37–40) — so the functions container gets the **same**
  `azure-monitor-opentelemetry` dep as the backend (§f). No functions-specific
  `requirements.txt` exists.
- No `OTEL_*` / `telemetryMode` / App-Insights env vars are baked into the image
  — the connection string is injected at runtime by Bicep (§c).

---

## (f) Dependencies + versions

Declared in `v2/pyproject.toml` `[project.dependencies]` (lines 69–71):

```toml
# Observability
"azure-monitor-opentelemetry>=1.6.10,<2.0",
"opentelemetry-instrumentation-httpx>=0.52b0",
```

Resolved / installed (from `v2/.venv` dist-info + `uv.lock`):

- `azure-monitor-opentelemetry` **1.8.7**
- `azure-monitor-opentelemetry-exporter` **1.0.0b51** (transitive; the actual
  App Insights exporter — parses `APPLICATIONINSIGHTS_CONNECTION_STRING`)

Both backend and functions containers install these (same `pyproject.toml`
source; §e Dockerfile note). Dependency availability is **not** a suspect.

---

## (g) Assessment — most likely root cause(s), ranked

### 1. (Backend, confirmed) Deployed revision predates the Bicep env-var rename — typed setting empty

The backend reads `AZURE_APP_INSIGHTS_CONNECTION_STRING`; the *originally*
deployed backend container was wired with the standard
`APPLICATIONINSIGHTS_CONNECTION_STRING`, so `ObservabilitySettings`
(`env_prefix="AZURE_"`) saw an empty string and `configure_azure_monitor` never
fired. This is the **root cause BUG-0055 itself confirms** for the backend half.
The Bicep now emits the correct `AZURE_`-prefixed name (`main.bicep` L1914), but
that only takes effect on `azd provision`. **If the live backend revision has not
been re-provisioned since the rename landed, backend telemetry is still off.**
Verification: `az containerapp show -g <RESOURCE_GROUP> -n ca-backend-<SUFFIX>`
and confirm the env var key is `AZURE_APP_INSIGHTS_CONNECTION_STRING` (not the
bare name) with a non-empty value.

### 2. (Both halves) Not-yet-provisioned / not-yet-cloud-verified deploy timing

BUG-0055 states both fixes are "code/config-complete but **not yet cloud-
verified**." The worker `configure_telemetry()` and the backend rename both
landed 2026-06-23 while deploys were blocked (Flex build outage). The function
later migrated to a Container App (BUG-0058, 2026-07-02) with a confirmed
`azd deploy function` — **but `azd deploy` pushes only the container image and
does NOT re-run Bicep env-var wiring.** Whether a full `azd provision` / `azd up`
ran to (a) apply the backend env-var rename and (b) create/refresh the functions
Container App with `APPLICATIONINSIGHTS_CONNECTION_STRING` is the central open
verification gap. If only image-level deploys happened, both containers may still
carry stale env-var wiring (or, for the freshly-migrated functions Container App,
the env var may be present if the migration itself was a provision — needs
checking).

### 3. (Functions, partial / by design) Host-level `requests` telemetry not enabled

Even once the worker exports `traces`/`dependencies`/`logs`, host-emitted
`requests` (invocation) telemetry requires the host's own App Insights /
OpenTelemetry integration. `host.json` has **no `telemetryMode`** → host OTel
off. For a containerized Functions host the classic integration reads
`APPLICATIONINSIGHTS_CONNECTION_STRING`, but BUG-0055 explicitly calls out host
`requests` as a separate follow-up. So a partial signal ("traces appear but
`requests` still 0") is expected and would **not** by itself mean the fix
failed. This also means the containerized-host assumption (BUG-0055 was written
for Flex) should be re-checked: a Container App running the Functions runtime may
handle host telemetry differently than Flex Consumption did.

### 4. (Environment-dependent) `enableMonitoring=false` ⇒ no App Insights, no env var

If a given environment deployed with `enableMonitoring=false`
(`ENABLE_MONITORING=false`), the `applicationInsights` module is skipped and
**neither** container receives the connection-string env var (both wrapped in
`enableMonitoring ? [...] : []`), guaranteeing zero telemetry. BUG-0063 records a
**postgresql** test deploy that ran with `ENABLE_MONITORING=false`. **However**,
for the environment BUG-0055 observed, `appi-<SUFFIX>` **exists** and its
connection string matches, so `enableMonitoring=true` there — rule this out for
the cosmosdb env, but confirm `ENABLE_MONITORING` per-environment before
concluding.

### Explicitly RULED OUT

- **`environment == production` / `AZURE_ENVIRONMENT` gating (BUG-0069).**
  Telemetry init is gated **only** on a non-empty connection string, never on
  `environment`. BUG-0069's unset-`AZURE_ENVIRONMENT` does **not** affect
  telemetry export. (§d, §h.)
- **`host.json` adaptive sampling dropping everything.** Adaptive sampling thins
  volume; it never yields absolute `[0, null, null]`, and it governs the host
  pipeline, not the worker's `configure_azure_monitor` OTel exporter. (§e.)
- **Missing dependency.** `azure-monitor-opentelemetry` 1.8.7 (+ exporter
  1.0.0b51) is installed in both containers. (§f.)

---

## (h) `environment` / `AZURE_ENVIRONMENT` cross-check (BUG-0069)

- `AppSettings.environment` — `v2/src/backend/core/settings.py` line 533:
  `environment: Environment = Environment.LOCAL` (default `local`).
- `Environment` StrEnum defined at `settings.py` line 41.
- The field's docstring (lines 517–533) says production deploys set
  `AZURE_ENVIRONMENT=production` via `main.bicep`, which "flips the final
  configuration to production — enabling real environment reporting and
  **disabling the local-dev identity bypass** used by chat." It does **not**
  mention telemetry.
- Grep confirms telemetry never branches on `environment`. So even if the
  deployed runtime reports `environment=local` (the BUG-0069 symptom), telemetry
  export is unaffected. This aligns with the user-memory note
  `config-defaults-dev-first.md` (dev-first defaults; prod flipped by IaC env
  vars) — telemetry deliberately does not participate in that gate.

---

## Exact-path index (for quick navigation)

| Concern | Path | Line(s) |
| --- | --- | --- |
| BUG-0055 detail | `v2/docs/bugs.md` | 998–1010 |
| Backend import | `v2/src/backend/app.py` | 25 |
| Backend init (gate) | `v2/src/backend/app.py` | 67–77 |
| Backend settings field | `v2/src/backend/core/settings.py` | 281–287 |
| Backend settings wire | `v2/src/backend/core/settings.py` | 551 |
| `environment` field | `v2/src/backend/core/settings.py` | 517–533 |
| Admin report (non-export) | `v2/src/backend/routers/admin.py` | 132 |
| Functions telemetry module | `v2/src/functions/core/telemetry.py` | 1–46 |
| Functions telemetry call | `v2/src/functions/function_app.py` | 19, 24–26 |
| host.json (sampling) | `v2/src/functions/host.json` | 1–21 |
| Bicep backend env var | `v2/infra/main.bicep` | 1904–1919 |
| Bicep functions env var | `v2/infra/main.bicep` | 2188–2197 |
| `enableMonitoring` default | `v2/infra/main.bicep` | 208 |
| App Insights module (gated) | `v2/infra/main.bicep` | 316 |
| Deps | `v2/pyproject.toml` | 69–71 |
| Functions Dockerfile deps | `v2/docker/Dockerfile.functions` | 37–40 |
| `.env.sample` note | `v2/.env.sample` | 69–76 |

---

## Clarifying questions (cannot be answered by code alone)

1. Has a full `azd provision` / `azd up` (not just `azd deploy`) run against the
   live cosmosdb environment **since 2026-06-23**, so the backend container's env
   var key is now `AZURE_APP_INSIGHTS_CONNECTION_STRING`? (Root cause #1/#2 hinge
   on this. Verify with `az containerapp show ... --query
   "properties.template.containers[0].env"`.)
2. On the live **functions Container App**, is `APPLICATIONINSIGHTS_CONNECTION_STRING`
   present and non-empty in the running revision's env? (The migration from Flex
   may or may not have carried the Bicep env wiring.)
3. Is `ENABLE_MONITORING=true` on the observed live environment? (`appi-<SUFFIX>`
   exists, implying yes, but confirm the app-settings value to fully rule out
   root cause #4.)
4. For the containerized Functions host, do we expect host-emitted `requests`
   telemetry without `host.json` `telemetryMode`, or is worker-only telemetry
   (`traces`/`dependencies`) acceptable for closing BUG-0055? (Determines whether
   #3 is a blocker or a documented follow-up.)

---

## Recommended next research (not completed this session)

- [ ] Read the two `test_app_lifespan.py` tests that assert the backend reads
      `AZURE_APP_INSIGHTS_CONNECTION_STRING` (confirm the drift-guard names).
- [ ] Read `tests/functions/core/test_telemetry.py` (configured + no-op paths)
      to confirm the worker contract matches the deployed behavior.
- [ ] Inspect ADR-0018 (Amendment 1) for the authoritative env-var-name split
      rationale and any host-telemetry follow-up decisions.
- [ ] Live: dump both containers' effective env vars (`az containerapp show`) to
      confirm the actual deployed connection-string keys/values.
- [ ] Confirm whether the containerized Functions host auto-injects/reads
      `APPLICATIONINSIGHTS_CONNECTION_STRING` the same way Flex did (docs /
      live check), since the BUG-0055 narrative predates the Container App
      migration.
