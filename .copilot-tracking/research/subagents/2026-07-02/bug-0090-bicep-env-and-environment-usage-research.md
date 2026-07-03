<!-- markdownlint-disable-file -->
# BUG-0090 research — Bicep env-var wiring for backend auth flags + `settings.environment` usage audit

Status: Complete
Date: 2026-07-02
Scope: research only, no code modified.

Supports an implementation plan that will (a) delete backend `require_admin_auth` +
the `requires_role` / Easy-Auth admin role gate, replacing it with a minimal
`get_user_id` that only validates `x-ms-client-principal-id` is a well-formed GUID;
(b) possibly retire the `environment` settings field + `Environment` enum.

---

## Gap 1 — Bicep / env-var wiring for `AZURE_REQUIRE_ADMIN_AUTH` and `AZURE_ENVIRONMENT`

### Summary table

| Env var | File:line | Container app | Value | Backed by param/var? |
| --- | --- | --- | --- | --- |
| `AZURE_ENVIRONMENT` | v2/infra/main.bicep:1805 | backend (`ca-backend-<suffix>`) | `'production'` (string literal) | No — inline literal |
| `AZURE_REQUIRE_ADMIN_AUTH` | v2/infra/main.bicep:1813 | backend (`ca-backend-<suffix>`) | `'false'` (string literal) | No — inline literal |
| `AZURE_ENVIRONMENT` | v2/infra/main.bicep:2160 | functions (`ca-func-<suffix>`) | `'production'` (string literal) | No — inline literal |
| `AZURE_REQUIRE_ADMIN_AUTH` | — (not set anywhere) | functions: NOT set; frontend: NOT set | — | — |

Both flags are **hard-coded inline string literals** in the container-app `env` blocks.
There is **no** `param requireAdminAuth`, `param environmentName`, `var environment`,
or any other Bicep declaration feeding their `value:`. Confirmed by:
- grep `param environmentName|param require|var environment|param environment` in main.bicep → 0 matches.

### Container-app module → symbolic name → line mapping

- `backendContainerApp` module declared at v2/infra/main.bicep:1724 → resource name `ca-backend-<suffix>` (`var` at v2/infra/main.bicep uses `backendContainerAppName`; see `frontendContainerAppName`/`functionContainerAppName` vars at lines 1554–1555). Its `env: union([...])` block spans ~1793–1925 and contains BOTH auth env entries.
- `frontendContainerApp` module declared at v2/infra/main.bicep:1945 → resource name `ca-frontend-<suffix>`. Its `env: [...]` block (v2/infra/main.bicep:2004–2011) contains **only** `BACKEND_API_URL`. Neither `AZURE_ENVIRONMENT` nor `AZURE_REQUIRE_ADMIN_AUTH` is present. Clean.
- `functionContainerApp` raw resource (`Microsoft.App/containerApps@2024-10-02-preview`, `kind: 'functionapp'`) declared at v2/infra/main.bicep:2046 → resource name `ca-func-<suffix>`. Its `env: union([...])` block (starts ~2140) contains `AZURE_ENVIRONMENT` at line 2160 but **no** `AZURE_REQUIRE_ADMIN_AUTH`.

### Exact Bicep snippets

Backend env block — `AZURE_ENVIRONMENT` at v2/infra/main.bicep:1805 (preceded by explanatory comment lines 1797–1805):

```bicep
            // Runtime mode (AppSettings.environment). Pinned to 'production'
            // on every cloud deploy so the runtime reports the real
            // environment (GET /api/admin/status) and DISABLES the local-dev
            // identity bypass used by chat: backend.dependencies.get_user_id
            // folds an anonymous caller into the synthetic 'local-dev'
            // partition ONLY when environment == 'local', so a deployed
            // runtime must never fall back to the 'local' default.
            //
            // This field no longer governs the admin auth WALL -- that is
            // controlled separately by AZURE_REQUIRE_ADMIN_AUTH below.
            { name: 'AZURE_ENVIRONMENT', value: 'production' }
```

Backend env block — `AZURE_REQUIRE_ADMIN_AUTH` at v2/infra/main.bicep:1813 (comment lines 1806–1812):

```bicep
            // Admin auth wall (AppSettings.require_admin_auth). 'false' (the
            // MACAE-faithful default) leaves /api/admin/* reachable without
            // Easy Auth claims. Set to 'true' to require Easy Auth admin-role
            // claims on admin routes; backend.dependencies.requires_role then
            // fails closed (401 without claims, 403 without the role). A
            // present claims blob is always role-checked regardless of this
            // value -- the flag relaxes the auth wall, never role enforcement.
            { name: 'AZURE_REQUIRE_ADMIN_AUTH', value: 'false' }
```

Functions env block — `AZURE_ENVIRONMENT` at v2/infra/main.bicep:2160 (comment lines 2158–2159):

```bicep
              // Runtime mode (AppSettings.environment) -- pin 'production' so the
              // deployed config reports production, parity with the backend.
              { name: 'AZURE_ENVIRONMENT', value: 'production' }
```

### Is `AZURE_ENVIRONMENT` consumed by the FUNCTIONS app?

- **Bicep:** YES — set on the functions container app at v2/infra/main.bicep:2160.
- **Functions runtime code:** NO. Grep of v2/src/functions/** for `settings.environment` / `Environment` / `require_admin_auth` / `AZURE_ENVIRONMENT` found **zero** functional reads — only two incidental prose uses of the word "environment":
  - v2/src/functions/batch_start/models.py:12 (docstring: "reads the container name from environment via").
  - v2/src/functions/core/parsers/document_intelligence_parser.py:105 (error string: "Set it in the ingestion runtime environment").
- The functions worker DOES construct the shared `AppSettings` (imports `backend.core.settings`), so `AZURE_ENVIRONMENT=production` merely sets the `environment` field on the functions' settings instance — a value the functions code never branches on.
- **Consequence for removal:** removing the `environment` field will NOT break the functions runtime. The bicep line 2160 (and 1805) would become dead no-ops. Safe to remove them too; not strictly required to (see extra-ignore note below).

### azd params / `.bicepparam` / `azure.yaml`

- v2/infra/main.parameters.json — grep `ENVIRONMENT|ADMIN_AUTH|environment|requireAdmin` → **0 matches**. Neither flag is an azd parameter.
- v2/azure.yaml — grep `ENVIRONMENT|ADMIN_AUTH|environment` → **0 matches**.
- No `.bicepparam` files exist under v2/** (file_search returned only v2/azure.yaml + v2/infra/main.parameters.json).
- v2/infra/main.json is the compiled ARM output of main.bicep (contains the same two literals at the compiled lines 834 / 48372). It is generated, not hand-authored.

### `model_config` extra-handling note (relevant to whether stray bicep env must be removed in lockstep)

`AppSettings.model_config` = `SettingsConfigDict(env_prefix="AZURE_", ..., extra="ignore")` at v2/src/backend/core/settings.py:509–513. Because `extra="ignore"`, leaving `AZURE_ENVIRONMENT` set in bicep after deleting the `environment` field will **not** raise a Pydantic `ValidationError` at settings load — the unknown env var is silently ignored. So removing the bicep lines is a cleanliness step, not a hard prerequisite for deleting the field.

---

## Gap 2 — Is `settings.environment` used by NON-auth code?

### Exhaustive `settings.environment` reads (exact-string grep over v2/src/**)

Only THREE runtime reads exist (all others are comments/docstrings):

| File:line | Auth? | What it does |
| --- | --- | --- |
| v2/src/backend/dependencies.py:375 | AUTH | `get_user_id`: `settings.environment is Environment.LOCAL` — part of `allow_open_auth` local-dev bypass. |
| v2/src/backend/dependencies.py:459 | AUTH | `requires_role._checker`: `settings.environment is Environment.LOCAL` — part of `allow_open_admin` local-dev bypass. |
| v2/src/backend/routers/admin.py:138 | **NON-AUTH** | `status_endpoint`: `environment=settings.environment` — populates the `AdminStatus.environment` field returned by `GET /api/admin/status`. |

Documentation-only mentions (NOT code, no action needed): settings.py:42 (enum docstring), dependencies.py:303 (comment), admin.py:35 (docstring), routers/history.py:12 (docstring).

`\.environment\b` (attribute-access) grep over v2/src/backend/** returned the same 3 runtime sites + the same 3 doc mentions — nothing else. No telemetry/observability, no CORS, no logging-level, no docs-enabled toggle reads `settings.environment`. The observability/CORS/logging setup does NOT branch on it.

### Answer: after auth branching is removed, is `environment` still referenced?

**YES — one non-auth consumer remains: v2/src/backend/routers/admin.py:138**, which surfaces the value into the `AdminStatus` response of `GET /api/admin/status`. This is a real operator-facing telemetry field ("what environment is this runtime reporting").

Therefore the `environment` field + `Environment` enum are **NOT trivially dead** after removing the two auth reads. To fully retire them, the plan must ALSO update the AdminStatus chain:

- v2/src/backend/routers/admin.py:138 — drop `environment=settings.environment,` from the `AdminStatus(...)` constructor.
- v2/src/backend/models/admin.py:113 — drop `environment: str` from the `AdminStatus` model (allow-list model; field is intentional, class docstring at models/admin.py:100–108).
- v2/src/frontend/src/models/admin.tsx:73 — drop `environment: string;` from the `AdminStatus` TS interface (mirrors the backend model; comment at line 67 "Mirrors `backend.models.admin.AdminStatus`").
- Tests asserting on `AdminStatus.environment` (e.g. `test_status_does_not_leak_sensitive_settings` referenced at models/admin.py:106; v2/tests/backend/test_admin.py) — will need the field expectation removed.

Frontend render note: v2/src/frontend/src/App.tsx:139 calls `getAdminStatus()` once at boot (App.tsx:16 docstring), but grep found **no** `.environment` read in App.tsx — the field is fetched into the typed `AdminStatus` body but is not rendered in any admin page today. So deleting it has no visible-UI regression; it's a type-shape change only.

**Bottom line for the plan:**
- Minimal-change path: remove ONLY the two auth reads (dependencies.py:375, :459) and their `Environment.LOCAL` branches; KEEP the `environment` field + enum as a decorative status-report value fed to admin.py:138. Field becomes harmless.
- Full-retirement path: delete field + enum AND update the 4 AdminStatus chain sites (admin.py:138, models/admin.py:113, models/admin.tsx:73, admin tests). Bicep lines 1805 + 2160 can be removed for cleanliness (not required due to `extra="ignore"`).

### `Environment` enum definition

- Defined at v2/src/backend/core/settings.py:41–52 (`class Environment(StrEnum)`).
- Members: `LOCAL = "local"` (line 51), `PRODUCTION = "production"` (line 52).
- Imported in v2/src/backend/dependencies.py:39 (`from backend.core.settings import AppSettings, Environment, get_settings`) — the only functional importer.
- Field declaration: `environment: Environment = Environment.LOCAL` at v2/src/backend/core/settings.py:533 (default = LOCAL, per the config-defaults-dev-first rule; comment at lines 521–532).

### `require_admin_auth` field + consumers (for the delete-the-flag half of the plan)

- Field declared at v2/src/backend/core/settings.py:543: `require_admin_auth: bool = False` (comment lines 535–542, default `False` = open posture).
- Runtime consumers (both in the two auth extractors): v2/src/backend/dependencies.py:375 (`not settings.require_admin_auth` in `get_user_id`'s `allow_open_auth`) and v2/src/backend/dependencies.py:459-ish (`not settings.require_admin_auth` in `requires_role._checker`'s `allow_open_admin`).
- No other production reader. Tests: v2/tests/backend/core/test_settings.py (`test_require_admin_auth_defaults_to_false`, `test_require_admin_auth_env_override_enables_wall`), v2/tests/backend/test_admin.py:75, v2/tests/backend/test_dependencies.py.

---

## Evidence / commands run

- grep `AZURE_REQUIRE_ADMIN_AUTH|AZURE_ENVIRONMENT` in v2/infra/** → main.bicep:1804(comment),1805,1813,2160 + main.json(compiled).
- grep `param environmentName|param require|var environment|param environment` in main.bicep → 0.
- grep `ENVIRONMENT|ADMIN_AUTH|environment|requireAdmin` in main.parameters.json → 0.
- grep `ENVIRONMENT|ADMIN_AUTH|environment` in v2/azure.yaml → 0.
- file_search `v2/**/{main.parameters.json,*.bicepparam,azure.yaml}` → only main.parameters.json + azure.yaml (no bicepparam).
- grep exact `settings.environment` in v2/src/** → 6 hits (3 code: deps:375, deps:459, admin:138; 3 doc).
- grep `\.environment\b` in v2/src/backend/** → same 6.
- grep `settings.environment|Environment|require_admin_auth|AZURE_ENVIRONMENT` in v2/src/functions/** → 2 incidental prose hits only.
- read settings.py:38–60 (enum), 515–560 (fields + model_config extra="ignore" at 509–513).
- read dependencies.py:295–470 (both auth extractors), admin.py:1–150 (status endpoint), models/admin.py:95–120 (AdminStatus).

## Clarifying questions

None blocking. One decision for the plan author (not a research gap): choose the
minimal-change path (keep `environment` as decorative status field) vs the
full-retirement path (delete field/enum + update the 4-site AdminStatus chain +
admin tests + optionally bicep lines 1805/2160).
