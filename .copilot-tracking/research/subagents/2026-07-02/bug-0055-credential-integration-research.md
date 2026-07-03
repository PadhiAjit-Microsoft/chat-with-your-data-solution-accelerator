# BUG-0055 — Credential integration research (App Insights zero telemetry fix)

Status: **Complete**
Scope: READ-ONLY. Answer credential-integration questions for wiring a
synchronous managed-identity `credential=` into `configure_azure_monitor`
on both runtimes (backend Container App + Functions worker).

Placeholder convention: any Azure ID is written as `<SUFFIX>` /
`<AZURE_UAMI_CLIENT_ID>` per repo Hard Rule #18.

---

## TL;DR answers to the return questions

- **Reusable SYNC managed-identity factory?** **No.** Every credentials
  provider under `v2/src/backend/core/providers/credentials/` returns an
  **ASYNC** credential (`azure.identity.aio.*`). There is no sync factory
  to reuse. The fix must construct
  `azure.identity.ManagedIdentityCredential(client_id=...)` **directly**
  (sync, top-level `azure.identity` namespace).
- **UAMI client-id settings field + env var:**
  `settings.identity.uami_client_id` (str, default `""`), env var
  **`AZURE_UAMI_CLIENT_ID`**. Defined in `IdentitySettings`
  (`v2/src/backend/core/settings.py:147`; class at line 137;
  `env_prefix="AZURE_"`). Composed onto `AppSettings.identity`
  (settings.py:545).
- **Backend telemetry-test patch target:**
  `backend.app.configure_azure_monitor`
  (monkeypatched at `v2/tests/backend/test_app_lifespan.py:454` and again
  in the empty-setting test ~line 495).
- **Functions telemetry-test patch target:**
  `functions.core.telemetry.configure_azure_monitor`
  (monkeypatched at `v2/tests/functions/core/test_telemetry.py:16` and
  line 34).
- **Existing telemetry test function names:**
  - Backend (`v2/tests/backend/test_app_lifespan.py`):
    - `test_lifespan_configures_app_insights_from_typed_settings` (line 418)
    - `test_lifespan_skips_app_insights_when_typed_setting_empty` (line 464)
  - Functions (`v2/tests/functions/core/test_telemetry.py`):
    - `test_configure_telemetry_noop_without_connection_string` (line 8)
    - `test_configure_telemetry_configures_when_connection_string_set` (line 22)

---

## (a) Credentials provider inventory — SYNC vs ASYNC

Directory: `v2/src/backend/core/providers/credentials/`
Files: `base.py`, `cli.py`, `managed_identity.py`, `registry.py`,
`_instance.py`, `__init__.py`.

| File | Registry key | Class | Returns | SYNC / ASYNC |
|------|--------------|-------|---------|--------------|
| `base.py` | (ABC) | `BaseCredentialProvider` | `AsyncTokenCredential` | **ASYNC** |
| `cli.py` | `@registry.register("cli")` | `CliCredentialProvider` | `azure.identity.aio.AzureCliCredential` | **ASYNC** |
| `managed_identity.py` | `@registry.register("managed_identity")` | `ManagedIdentityCredentialProvider` | `azure.identity.aio.DefaultAzureCredential(managed_identity_client_id=...)` | **ASYNC** |
| `registry.py` | — | holds `Registry` + `select_default()` helper | — | — |

### base.py (full)

`v2/src/backend/core/providers/credentials/base.py`

```python
from abc import ABC, abstractmethod
from azure.core.credentials_async import AsyncTokenCredential
from backend.core.settings import AppSettings

class BaseCredentialProvider(ABC):
    def __init__(self, settings: AppSettings) -> None:
        self._settings = settings

    @abstractmethod
    async def get_credential(self) -> AsyncTokenCredential:
        ...
```

- Line 8: `from azure.core.credentials_async import AsyncTokenCredential`
  → the ABC contract is **async-only**.

### cli.py (full)

`v2/src/backend/core/providers/credentials/cli.py`

```python
from azure.identity.aio import AzureCliCredential          # line 11 (ASYNC)
from .base import BaseCredentialProvider
from .registry import registry

@registry.register("cli")                                   # line 15
class CliCredentialProvider(BaseCredentialProvider):
    async def get_credential(self) -> AzureCliCredential:
        return AzureCliCredential()
```

### managed_identity.py (full)

`v2/src/backend/core/providers/credentials/managed_identity.py`

```python
from azure.identity.aio import DefaultAzureCredential       # line 15 (ASYNC)
from .base import BaseCredentialProvider
from .registry import registry

@registry.register("managed_identity")                      # line 19
class ManagedIdentityCredentialProvider(BaseCredentialProvider):
    async def get_credential(self) -> DefaultAzureCredential:
        client_id = self._settings.identity.uami_client_id or None   # line 22
        return DefaultAzureCredential(managed_identity_client_id=client_id)
```

- Reads the client id from `self._settings.identity.uami_client_id or None`
  (line 22). Passes it to the **async** `DefaultAzureCredential` via
  `managed_identity_client_id=`.

### registry.py — `select_default` heuristic

`v2/src/backend/core/providers/credentials/registry.py`

```python
def select_default(uami_client_id: str | None) -> str:      # line ~40
    return "managed_identity" if uami_client_id else "cli"
```

- `_instance.py` holds the `Registry[...]` instance (imported here as
  `registry`). Eager side-effect imports of `cli`, `managed_identity`
  fire the `@registry.register(...)` decorators. Entry-point plugins
  loaded via `load_entry_points("cwyd.providers.credentials")`.

**Conclusion:** no SYNC credential anywhere in this domain. The fix cannot
reuse a provider; it must instantiate the sync
`azure.identity.ManagedIdentityCredential(client_id=...)` at (or near) each
`configure_azure_monitor` call site.

---

## (b) How the UAMI client id is obtained at runtime

- **Settings field:** `settings.identity.uami_client_id`
- **Env var:** `AZURE_UAMI_CLIENT_ID`
- **Definition:** `IdentitySettings` — `v2/src/backend/core/settings.py`

```python
class IdentitySettings(BaseSettings):                       # line 137
    """Reads: AZURE_TENANT_ID, AZURE_UAMI_CLIENT_ID,
    AZURE_UAMI_PRINCIPAL_ID, AZURE_UAMI_RESOURCE_ID."""
    model_config = SettingsConfigDict(env_prefix="AZURE_", extra="ignore")
    tenant_id: str = ""
    uami_client_id: str = ""                                # line 147
    uami_principal_id: str = ""
    uami_resource_id: str = ""
```

- **AppSettings composition** (`v2/src/backend/core/settings.py`):
  - `identity: IdentitySettings = Field(default_factory=IdentitySettings)` — line 545
  - `observability: ObservabilitySettings = Field(default_factory=ObservabilitySettings)` — line 551
  - `class AppSettings(BaseSettings)` — line 506
  - `get_settings()` `@lru_cache(maxsize=1)` singleton — line 569

- **Observability connection-string field** (for reference — the string
  side of the fix): `settings.observability.app_insights_connection_string`,
  env var `AZURE_APP_INSIGHTS_CONNECTION_STRING`, defined in
  `ObservabilitySettings` at `v2/src/backend/core/settings.py:281`:

```python
class ObservabilitySettings(BaseSettings):                  # line 281
    model_config = SettingsConfigDict(env_prefix="AZURE_", extra="ignore")
    app_insights_connection_string: str = ""
    log_level: str = "INFO"
```

- **No `settings.credentials.client_id` / `settings.azure_client_id`
  exists.** The single carrier is `settings.identity.uami_client_id`.
  There is no bare `AZURE_CLIENT_ID` settings field either (grep across
  `v2/src` found no `AZURE_CLIENT_ID` field; only `AZURE_UAMI_CLIENT_ID`
  via the `AZURE_` prefix + `uami_client_id`).

- **Functions blueprints** obtain the same client id the same way:
  `credentials_registry.select_default(settings.identity.uami_client_id)`
  then `get_credential()` (async) — e.g.
  `v2/src/functions/blob_event/blueprint.py:98-101`. But
  `functions/core/telemetry.py` has **no** access to settings/credential
  today (see section d).

---

## (c) Backend `app.py` lifespan — full listing + reusable-credential analysis

File: `v2/src/backend/app.py`

### Top imports (lines 20-36)

```python
import logging                                              # 20
from contextlib import asynccontextmanager                 # 21
from typing import Any, AsyncGenerator                      # 22

from azure.ai.contentsafety.aio import ContentSafetyClient  # 24
from azure.core.credentials_async import AsyncTokenCredential  # 25... (see below)
from azure.monitor.opentelemetry import configure_azure_monitor  # pyright: ignore[reportUnknownVariableType]
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from backend.exception_handlers import install_exception_handlers
from backend.routers import admin, conversation, files, health, history, speech
from backend.core.providers.agents import registry as agents_registry
from backend.core.providers.credentials import registry as credentials_registry
from backend.core.providers.databases import registry as databases_registry
from backend.core.providers.llm import registry as llm_registry
from backend.core.providers.search import registry as search_registry
from backend.core.settings import AppSettings, IndexStore, NetworkSettings, get_settings
```

- `configure_azure_monitor` is a **top-level import** (line 25 region) —
  Hard Rule #17 compliant. `credentials_registry` is already imported at
  the top (available to the telemetry block if reordered).
- `AsyncTokenCredential` is imported (used by `_init_content_safety_client`).
  Note: the sync `ManagedIdentityCredential` / `TokenCredential`
  (`azure.core.credentials`) is **not** currently imported — the fix
  would add `from azure.identity import ManagedIdentityCredential`.

### `_lifespan` — telemetry block runs FIRST, credential built AFTER

`v2/src/backend/app.py` (lifespan starts ~line 66):

```python
@asynccontextmanager
async def _lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    settings = get_settings()

    conn_str = settings.observability.app_insights_connection_string.strip()
    if conn_str:
        configure_azure_monitor(connection_string=conn_str)   # <-- CALL SITE 1 (no credential=)
        logger.info("Application Insights telemetry configured.")
    else:
        logger.info(
            "AZURE_APP_INSIGHTS_CONNECTION_STRING not set; telemetry disabled."
        )

    cred_key = credentials_registry.select_default(settings.identity.uami_client_id)
    cred_provider = credentials_registry.registry.get(cred_key)(settings=settings)
    credential = await cred_provider.get_credential()         # <-- ASYNC credential, built AFTER telemetry
    llm_provider = llm_registry.registry.get("foundry_iq")(
        settings=settings, credential=credential
    )
    # ... agents_provider, database_client, runtime_overrides,
    #     search_provider, content_safety_client ...
    app.state.credential_provider = cred_provider
    app.state.credential = credential
    # ...
    try:
        yield
    finally:
        # reverse-order teardown; credential closed last:
        try:
            await credential.close()
        except Exception:  # noqa: BLE001
            logger.exception("Error closing Azure credential.")
```

### Reusable-credential verdict

- The lifespan-constructed `credential` (`app.state.credential`) is an
  **async** `AsyncTokenCredential` (aio `DefaultAzureCredential` /
  `AzureCliCredential`). `configure_azure_monitor` requires a **sync**
  `azure.core.credentials.TokenCredential` — so it **cannot** be reused.
- It is also constructed **after** the `configure_azure_monitor` call, so
  even ignoring sync/async it is not in scope at CALL SITE 1.
- **Recommended pattern:** build a short-lived sync
  `ManagedIdentityCredential(client_id=settings.identity.uami_client_id)`
  inside the `if conn_str:` block and pass it as `credential=`. Guard on
  `settings.identity.uami_client_id` being non-empty (local/dev has no
  UAMI — matches `select_default`'s own heuristic). The sync MI credential
  is cheap and holds no aiohttp session, so it does not need lifespan
  teardown the way the async ones do (the Azure Monitor exporter owns its
  token refresh lifecycle).

---

## (d) Functions `telemetry.py` (full) + `function_app.py` (head)

### `v2/src/functions/core/telemetry.py` — every line

```python
"""Pillar: Stable Core
Phase: 6 (Functions blueprints / modular RAG indexing pipeline)

Azure Monitor / OpenTelemetry export for the Functions worker.
... (docstring; host provides APPLICATIONINSIGHTS_CONNECTION_STRING,
     backend uses AZURE_-prefixed name per ADR 0018) ...
"""

import logging
import os

from azure.monitor.opentelemetry import configure_azure_monitor  # pyright: ignore[reportUnknownVariableType]

logger = logging.getLogger(__name__)

_APPLICATIONINSIGHTS_CONNECTION_STRING = "APPLICATIONINSIGHTS_CONNECTION_STRING"


def configure_telemetry() -> bool:
    conn_str = os.environ.get(_APPLICATIONINSIGHTS_CONNECTION_STRING, "").strip()
    if not conn_str:
        logger.info(
            "APPLICATIONINSIGHTS_CONNECTION_STRING not set; "
            "function telemetry disabled."
        )
        return False
    configure_azure_monitor(connection_string=conn_str)   # <-- CALL SITE 2 (no credential=)
    logger.info("Application Insights telemetry configured for functions.")
    return True
```

Key facts:
- Reads the **host** env var `APPLICATIONINSIGHTS_CONNECTION_STRING`
  (NOT the `AZURE_`-prefixed backend name — ADR 0018).
- **No settings import, no credential access today.** It only reads one
  env var and calls `configure_azure_monitor(connection_string=...)`.
- To pass `credential=`, the functions worker needs the UAMI client id.
  Two options: (1) read `os.environ.get("AZURE_CLIENT_ID")` /
  `os.environ.get("AZURE_UAMI_CLIENT_ID")` directly (keeps the module
  settings-free, matching its current thin style), or (2) import
  `backend.core.settings.get_settings().identity.uami_client_id`. The
  module is currently deliberately `os`-only; option (1) is the smaller
  change and preserves the "thin" convention noted in its docstring.
  (Confirm which env var the Functions app actually receives from infra —
  see the companion bicep-wiring research doc.)

### `v2/src/functions/function_app.py` (lines 1-40)

```python
"""Pillar: Stable Core / Phase: 6 — Modular RAG indexing pipeline host."""

import azure.functions as func
from pydantic import BaseModel, ConfigDict

from functions.add_url.blueprint import bp as add_url_bp
from functions.batch_push.blueprint import bp as batch_push_bp
from functions.batch_start.blueprint import bp as batch_start_bp
from functions.blob_event.blueprint import bp as blob_event_bp
from functions.core.telemetry import configure_telemetry
from functions.search_skill.blueprint import bp as search_skill_bp

# Wire Azure Monitor export before registering functions (no-op when the
# App Insights connection string is absent).
configure_telemetry()                                      # <-- module-import-time call

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)
app.register_functions(batch_start_bp)
# ... register batch_push_bp, add_url_bp, blob_event_bp, search_skill_bp ...
```

- `configure_telemetry()` is invoked at **module import time** (before any
  blueprint registration). Any credential logic added to
  `configure_telemetry` runs during cold start, before the host wires
  the async credential providers — so a **sync** credential is the only
  workable option here too.

### Functions credential wiring today (for context)

- Blueprints resolve credentials via the registry, e.g.
  `v2/src/functions/blob_event/blueprint.py:98-101`:

```python
cred_provider = credentials_registry.registry.get(
    credentials_registry.select_default(settings.identity.uami_client_id)
)(settings=settings)
async with await cred_provider.get_credential() as credential:
    ...
```

- All functions credential usage is **async** and per-invocation; the
  telemetry module is the only place that needs a **sync** credential.

---

## (e) Existing backend lifespan telemetry tests

File: `v2/tests/backend/test_app_lifespan.py`

### Shared fixtures / helpers

- `COSMOS_ENV` dict — line 13 (base env; **does not** set
  `AZURE_UAMI_CLIENT_ID`, so `select_default` → `"cli"` path).
- `_apply_env(monkeypatch, env)` — line 36. Explicitly `delenv`s
  `AZURE_UAMI_CLIENT_ID` (line ~38 in the clear-list) before applying the
  test env. Relevant: a fix asserting `credential=` in the "configures"
  test will need the env to include `AZURE_UAMI_CLIENT_ID` so a MI
  credential is actually constructed (otherwise the guard skips it).
- `_patched_lifespan(monkeypatch)` — line 49. Stubs
  credentials/llm/databases/agents registries so lifespan runs offline.
  Notably patches `backend.app.credentials_registry.select_default`
  (line 73) and `backend.app.credentials_registry.registry` (line 76).

### Test 1 — `test_lifespan_configures_app_insights_from_typed_settings`

- Definition: line **418**.
- Sets env `AZURE_APP_INSIGHTS_CONNECTION_STRING` = `InstrumentationKey=...;IngestionEndpoint=...`.
- Patches the search registry, then patches telemetry:

```python
captured: dict[str, str] = {}

def _fake_configure(*, connection_string: str) -> None:     # ~line 448
    captured["connection_string"] = connection_string

monkeypatch.setattr(
    "backend.app.configure_azure_monitor", _fake_configure   # line 454
)

app = create_app()
async with app.router.lifespan_context(app):
    pass

assert captured["connection_string"].startswith("InstrumentationKey=")  # ~line 462
```

- **Patch target:** `backend.app.configure_azure_monitor`.
- **`_fake_configure` signature is keyword-only `*, connection_string: str`.**
  A fix adding `credential=` to the real call would make this stub raise
  `TypeError: unexpected keyword argument 'credential'`. The stub must be
  widened (e.g. `def _fake_configure(*, connection_string, credential=None)`
  or `def _fake_configure(**kw)`), and a new assert added
  (`assert captured.get("credential") is not None`). To make a credential
  actually get built, add `AZURE_UAMI_CLIENT_ID` to this test's `env`.

### Test 2 — `test_lifespan_skips_app_insights_when_typed_setting_empty`

- Definition: line **464**.
- Deliberately sets the **legacy** `APPLICATIONINSIGHTS_CONNECTION_STRING`
  and `delenv`s `AZURE_APP_INSIGHTS_CONNECTION_STRING` to prove the legacy
  alias is ignored.
- Patches telemetry with a call counter:

```python
called = {"count": 0}

def _fake_configure(**_kw) -> None:                          # ~line 491
    called["count"] += 1

monkeypatch.setattr(
    "backend.app.configure_azure_monitor", _fake_configure   # ~line 495
)

app = create_app()
async with app.router.lifespan_context(app):
    pass

assert called["count"] == 0                                  # ~line 502
```

- **Patch target:** `backend.app.configure_azure_monitor`.
- This stub already uses `**_kw`, so it is forward-compatible with a
  `credential=` addition (it never reaches the call anyway — conn str empty).

---

## (f) Existing functions telemetry tests

File: `v2/tests/functions/core/test_telemetry.py` (whole file is 2 tests)

### Test 1 — `test_configure_telemetry_noop_without_connection_string`

- Definition: line **8**.

```python
monkeypatch.delenv("APPLICATIONINSIGHTS_CONNECTION_STRING", raising=False)
calls: list[str] = []
monkeypatch.setattr(
    "functions.core.telemetry.configure_azure_monitor",          # line 16 (patch target)
    lambda **kw: calls.append(kw["connection_string"]),
)
assert configure_telemetry() is False
assert calls == []
```

### Test 2 — `test_configure_telemetry_configures_when_connection_string_set`

- Definition: line **22**.

```python
conn = "InstrumentationKey=...;IngestionEndpoint=https://uksouth.in.applicationinsights.azure.com/"
monkeypatch.setenv("APPLICATIONINSIGHTS_CONNECTION_STRING", conn)
captured: dict[str, str] = {}
monkeypatch.setattr(
    "functions.core.telemetry.configure_azure_monitor",          # line 34 (patch target)
    lambda **kw: captured.update(kw),
)
assert configure_telemetry() is True
assert captured["connection_string"] == conn
```

- **Patch target (both):** `functions.core.telemetry.configure_azure_monitor`.
- Both stubs accept `**kw`, so a `credential=` addition flows into `kw`
  automatically. Test 2 can be extended with
  `assert captured.get("credential") is not None` once the fix passes a
  sync credential (and the test sets `AZURE_CLIENT_ID` /
  `AZURE_UAMI_CLIENT_ID` so a credential is actually constructed).

---

## (g) Existing `ManagedIdentityCredential` usage + `azure-identity` confirmation

- **Sync `azure.identity.ManagedIdentityCredential` usage in `v2/src/**`:
  NONE.** Grep for `from azure.identity import` and `ManagedIdentityCredential(`
  across `v2/src/**` returned **zero** matches. All credential code uses
  the **async** namespace `azure.identity.aio` (`DefaultAzureCredential`,
  `AzureCliCredential`). There is no established sync construction
  pattern — the fix introduces the first one.

- **`azure-identity` dependency — CONFIRMED.**
  `v2/pyproject.toml:48`:

  ```toml
  "azure-identity>=1.25,<2.0",
  ```

  The sync `ManagedIdentityCredential` ships in the same package under the
  top-level `azure.identity` namespace (the async variant used today is
  `azure.identity.aio`). No new dependency is required.

- **`azure-monitor-opentelemetry` — CONFIRMED.**
  `v2/pyproject.toml:70`:

  ```toml
  "azure-monitor-opentelemetry>=1.6.10,<2.0",
  ```

- **Construction pattern the fix should introduce** (no existing precedent
  to copy — this is the new sync form):

  ```python
  from azure.identity import ManagedIdentityCredential  # sync, azure.identity (NOT .aio)

  credential = ManagedIdentityCredential(client_id=settings.identity.uami_client_id)
  configure_azure_monitor(connection_string=conn_str, credential=credential)
  ```

  `configure_azure_monitor`'s `credential=` param accepts a sync
  `azure.core.credentials.TokenCredential`; `ManagedIdentityCredential`
  satisfies it. `client_id=` pins the specific UAMI (the same value the
  async provider passes as `managed_identity_client_id=`). Guard on
  `settings.identity.uami_client_id` being non-empty so local/dev
  (no UAMI, connection string usually empty anyway) does not attempt MI.

---

## Cross-file call-site summary (the two edits the fix targets)

| # | Runtime | File | Call today | Client-id source available in scope |
|---|---------|------|------------|--------------------------------------|
| 1 | Backend Container App | `v2/src/backend/app.py` `_lifespan` (telemetry block, ~line 69-79) | `configure_azure_monitor(connection_string=conn_str)` | `settings.identity.uami_client_id` (settings already fetched) |
| 2 | Functions worker | `v2/src/functions/core/telemetry.py` `configure_telemetry()` (~line 41) | `configure_azure_monitor(connection_string=conn_str)` | needs to read env (`AZURE_CLIENT_ID`/`AZURE_UAMI_CLIENT_ID`) or import `get_settings()` — module is `os`-only today |

Open confirmation (out of this doc's scope; defer to bicep-wiring research):
which client-id env var the **Functions app** actually receives from infra
(`AZURE_CLIENT_ID` vs `AZURE_UAMI_CLIENT_ID`), and whether the async
credential providers on the Functions side already rely on
`settings.identity.uami_client_id` being populated in that runtime.

---

## Recommended next research (not done here)

- [ ] Confirm the exact `credential=` parameter name/type on the pinned
  `azure-monitor-opentelemetry>=1.6.10` `configure_azure_monitor`
  signature (verify it is `credential`, sync `TokenCredential`).
- [ ] Confirm which UAMI client-id env var is injected into the **Functions**
  app by `v2/infra/main.bicep` (drives the section-d option 1 vs 2 choice).
- [ ] Confirm whether a short-lived sync `ManagedIdentityCredential` needs
  explicit `.close()` teardown, or whether the Azure Monitor exporter owns
  its lifecycle (affects whether the backend lifespan `finally` block must
  also close the telemetry credential).
