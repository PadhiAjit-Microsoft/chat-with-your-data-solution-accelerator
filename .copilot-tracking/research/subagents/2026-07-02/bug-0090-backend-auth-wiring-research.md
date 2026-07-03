<!-- markdownlint-disable-file -->
# BUG-0090 — Backend auth wiring research (`/api/admin/status` 401)

Status: Complete

Scope: Research-only. No production code was modified. This document maps the full
`user_id` / authentication path from HTTP request → route handler for the CWYD v2
backend, and pins the exact origin of the HTTP 401 on `GET /api/admin/status`.

Research target: `c:\workstation\Microsoft\github\cwyd-cdb\v2`

---

## TL;DR — precise root cause

The 401 on `GET /api/admin/status` is produced by the **admin role gate**, not by any
`user_id`-presence check. The admin routes attach `AdminUserIdDep`
(`= Depends(requires_role("admin"))`), and `requires_role("admin")._checker` raises
`401` at:

- `v2/src/backend/dependencies.py:471-478` — the `if not claims_raw:` branch, raising
  `HTTPException(status_code=401, detail="Missing client principal claims; Easy Auth claims header required.")`.

That exact detail string is the one quoted in the BUG-0090 registry row.

The 401 fires only when `allow_open_admin` is **False**, computed at
`v2/src/backend/dependencies.py:458-461`:

```python
allow_open_admin = (
    settings.environment is Environment.LOCAL
    or not settings.require_admin_auth
)
```

So the gate fails closed (401) only when **both**: `environment != LOCAL` (i.e.
`production`) **and** `require_admin_auth is True`. The SPA's admin API client forwards
only `x-ms-client-principal-id` (via `userIdHeaders()`), never the base64
`x-ms-client-principal` **claims** blob, so `claims_raw` is empty → 401.

The backend currently does **much more** than the user's desired behavior (check a
`user_id` header is present + is a valid GUID): it decodes an Easy Auth base64 JSON
claims blob, extracts role claims, checks for the `"admin"` role, and raises 401/403.
There is **no GUID/UUID format validation** of `user_id` anywhere in the backend.

---

## Q1 — BUG-0090 full entry (verbatim)

Source: `v2/docs/bugs.md`. There is **no `### BUG-0090` Details subsection** — only the
registry row exists (grep for `### BUG-0090` returns empty; grep for `BUG-0090` matches
only the registry table at `v2/docs/bugs.md:149`). The registry row (line 149) reads:

> | BUG-0090 | 2026-06-25 |  | infra | high | open | The **production admin panel is
> unreachable** — `GET /api/admin/status` on the public backend FQDN returns
> `401 {"detail":"Missing client principal claims; Easy Auth claims header required."}`
> for every caller (unauthenticated *and* the signed-in SPA). Found 2026-06-25
> immediately after BUG-0089 closed (production auth enforcement restored). **This 401
> is the auth gate working as designed**, not a regression: in `production`,
> `backend.dependencies.requires_role("admin")` requires the base64
> `x-ms-client-principal` Easy Auth claims blob carrying an `admin` role
> (dependencies.py). **Root cause: no Easy Auth identity source feeds the backend.** The
> backend Container App (`ca-backend-<SUFFIX>`) has **no** `authConfig` —
> `az containerapp auth show` returns `{}` — so Container Apps Easy Auth never injects
> the claims blob. The SPA calls the backend **directly** cross-origin
> (`frontend_app.py` serves the SPA + `/config` only; it does **not** proxy `/api/*` and
> cannot forge the trusted claims header), and no Entra app registration defining an
> `admin` app role exists in the deployment. The frontend App Service
> (`app-frontend-<SUFFIX>`) has Easy Auth enabled but its Entra provider reads back
> unconfigured (empty `clientId`/issuer), so the identity layer was never fully
> provisioned. Admin previously *appeared* to work **only** via the local-dev bypass on
> the stale revision (the BUG-0089 anomaly). Non-admin chat/history still works because
> the SPA forwards a default `x-ms-client-principal-id` that `get_user_id` accepts in
> production. **Fix (pending decision):** wire a real identity source — either **(A)**
> enable Container Apps Easy Auth on the backend behind an Entra app registration with
> an `admin` app role, or **(B)** make `frontend_app.py` reverse-proxy `/api/*` and
> reuse the frontend's Easy Auth (define the `admin` role on the frontend app
> registration). Both are structural infra + Entra chan… *(row truncated in source at
> the 2000-char read boundary; the visible portion ends mid-word "chan[ges]")*

Recorded metadata:

- **ID:** BUG-0090
- **Found:** 2026-06-25
- **Fixed:** (blank — still open)
- **Area:** infra
- **Severity:** high
- **Status:** open
- **Recorded root cause:** "no Easy Auth identity source feeds the backend" — the
  backend Container App has no `authConfig`, so the trusted claims blob is never
  injected; the split-host SPA calls the backend cross-origin and cannot forge the
  claims header; no Entra app registration with an `admin` role exists.
- **Referenced files:** `dependencies.py` (the `requires_role("admin")` gate),
  `frontend_app.py` (serves SPA + `/config`, does not proxy `/api/*`),
  `ca-backend-<SUFFIX>` / `app-frontend-<SUFFIX>` (infra).

Related registry rows (context):

- **BUG-0089** (`v2/docs/bugs.md:148`, fixed 2026-06-25) — the backend was reading
  `environment=local` on a stale revision, which activated the **local-dev admin
  bypass**, so `/api/admin/*` was reachable unauthenticated. Closing BUG-0089 restored
  production enforcement, which immediately surfaced BUG-0090.
- **BUG-0091** (`v2/docs/bugs.md:150`, fixed 2026-06-29) — removed the frontend
  `auth_enforced` health flag + `AuthBlocked` wall. Explicitly notes:
  "`require_admin_auth` (`AZURE_REQUIRE_ADMIN_AUTH`), `get_user_id`, and `requires_role`
  are **retained unchanged** as the orthogonal opt-in server-side admin-write gate."

---

## Q2 — Admin router

File: `v2/src/backend/routers/admin.py`

- Router object (line ~103):

  ```python
  router = APIRouter(prefix="/api/admin", tags=["admin"])
  ```

  There is **no** `dependencies=[...]` on the `APIRouter(...)` constructor — the auth
  gate is applied per-route via a dependency parameter, not router-wide.

- `GET /api/admin/status` route (lines 114-140):

  ```python
  @router.get("/status", response_model=AdminStatus)
  async def status_endpoint(
      settings: SettingsDep,
      overrides: RuntimeOverridesDep,
      _user: AdminUserIdDep,   # <-- the auth gate; return value discarded
  ) -> AdminStatus:
  ```

  The `_user: AdminUserIdDep` parameter is the admin auth guard. Its returned user id is
  intentionally discarded (`_user`) — on the status route the dependency exists **only**
  to enforce the gate.

- Every `/api/admin/*` route depends on the same gate. Confirmed dependencies attached:
  - `status_endpoint` → `_user: AdminUserIdDep` (admin/status).
  - `config_endpoint` (`GET /config`) → `_user: AdminUserIdDep`.
  - `config_effective_endpoint` (`GET /config/effective`) → `_user: AdminUserIdDep`.
  - The PATCH/upload/reprocess/ingest/delete routes further down the file also take
    `AdminUserIdDep` (imported at `v2/src/backend/routers/admin.py:59`).

- Other DI on these routes (non-auth): `SettingsDep`, `RuntimeOverridesDep`,
  `SearchProviderDep`, `AgentsProviderDep`, `CredentialDep`, `DatabaseClientDep`,
  `RuntimeOverridesDep` — all imported from `backend.dependencies`
  (`v2/src/backend/routers/admin.py:58-66`).

- Admin-specific auth guard: **yes** — `AdminUserIdDep`, which is
  `Annotated[str, Depends(REQUIRE_ADMIN_USER)]` where
  `REQUIRE_ADMIN_USER = requires_role("admin")`. This is the role-gated variant, distinct
  from the lighter `UserIdDep` used by chat/history.

The router module docstring (`v2/src/backend/routers/admin.py:29-46`) documents the gate
explicitly: reads `x-ms-client-principal` + `x-ms-client-principal-id`, returns the
Entra object id when the `admin` role claim is present, raises **401** when Easy Auth is
missing/malformed in production, **403** when authenticated but lacking the role, and
falls back to `"local-dev"` in `environment == "local"`.

---

## Q3 — Auth dependency / user_id extraction

All auth wiring lives in one file: `v2/src/backend/dependencies.py`. There is **no
middleware** enforcing auth globally (see "No global middleware" below). Two sibling
extractors share the same Easy Auth surface:

### 3a. `get_user_id` — the lighter, chat/history extractor (NOT role-gated)

`v2/src/backend/dependencies.py:346-384`:

```python
def get_user_id(request: Request, settings: SettingsDep) -> str:
    value = request.headers.get(_PRINCIPAL_ID_HEADER, "").strip()
    if value:
        if not _is_valid_principal_id(value):
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Malformed client principal id.",
            )
        return value
    allow_open_auth = (
        settings.environment is Environment.LOCAL
        or not settings.require_admin_auth
    )
    if allow_open_auth:
        return _LOCAL_DEV_USER
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Missing client principal; Easy Auth header required.",
    )
```

- Header it needs: `x-ms-client-principal-id` (`_PRINCIPAL_ID_HEADER`, defined at
  `v2/src/backend/dependencies.py:319`).
- If the header **is present**: validates it against `_is_valid_principal_id` (a broad
  character allowlist — NOT a GUID check; see Q4), returns it verbatim. Malformed →
  **401** at line 369-373 (`"Malformed client principal id."`).
- If the header **is absent**: folds to the synthetic `"local-dev"` partition when
  `allow_open_auth` is True (`environment is LOCAL` OR `require_admin_auth is False`),
  else raises **401** at line 379-383 (`"Missing client principal; Easy Auth header
  required."`).
- Does **not** parse the base64 `x-ms-client-principal` claims blob at all — it only
  reads the `-id` header.
- Exposed as `UserIdDep = Annotated[str, Depends(get_user_id)]`
  (`v2/src/backend/dependencies.py:387`). Consumed by `history.py` and
  `conversation.py`.

### 3b. `requires_role(role)` — the admin role gate (produces the BUG-0090 401)

`v2/src/backend/dependencies.py:433-510`. The returned `_checker` (line 446):

```python
def _checker(request: Request, settings: SettingsDep) -> str:
    principal_id = request.headers.get(_PRINCIPAL_ID_HEADER, "").strip()
    claims_raw = request.headers.get(_PRINCIPAL_HEADER, "").strip()

    allow_open_admin = (                              # lines 458-461
        settings.environment is Environment.LOCAL
        or not settings.require_admin_auth
    )

    if not claims_raw:                                # lines 467-478
        if allow_open_admin:
            return _LOCAL_DEV_USER
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=(
                "Missing client principal claims; "
                "Easy Auth claims header required."
            ),
        )

    principal = _decode_easy_auth_principal(claims_raw)
    if principal is None:                             # lines 480-484
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Malformed client principal payload.",
        )

    roles = _extract_roles(principal)
    if role not in roles:                             # lines 486-490
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=f"Role '{role}' required to access this resource.",
        )

    if principal_id:                                  # lines 494-502
        return principal_id
    if allow_open_admin:
        return _LOCAL_DEV_USER
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Missing client principal id header.",
    )
```

- Headers it needs: `x-ms-client-principal` (`_PRINCIPAL_HEADER`, base64 JSON claims
  blob, defined at `v2/src/backend/dependencies.py:320`) **and** optionally
  `x-ms-client-principal-id`.
- **The BUG-0090 401**: line 471-478. When `claims_raw` is empty (the SPA never sends
  the claims blob) and `allow_open_admin` is False (`production` + `require_admin_auth`
  True), it raises exactly `detail="Missing client principal claims; Easy Auth claims
  header required."` — the string quoted in the BUG-0090 registry row.
- Other raises: malformed base64/JSON claims → **401** (line 482); authenticated but
  missing the `"admin"` role → **403** (line 488); valid claims + role but no principal
  id header and wall on → **401** (line 501).
- Easy Auth header parsing:
  - `_decode_easy_auth_principal` (`v2/src/backend/dependencies.py:389-406`) base64-decodes
    + JSON-parses the `x-ms-client-principal` header, returning `None` on any decode
    failure (which the caller maps to 401).
  - `_extract_roles` (`v2/src/backend/dependencies.py:408-431`) walks
    `principal["claims"]` and collects `val` where `typ` is either `"roles"`
    (`_ROLE_TYP_SHORT`) or the full schema URI
    (`http://schemas.microsoft.com/ws/2008/06/identity/claims/role`, `_ROLE_TYP_FULL`).
- If the claims headers are absent: in production-with-wall-on it 401s; in
  local/wall-off it returns `"local-dev"`.
- Cached singleton + typed alias (`v2/src/backend/dependencies.py:513-514`):

  ```python
  REQUIRE_ADMIN_USER = requires_role("admin")
  AdminUserIdDep = Annotated[str, Depends(REQUIRE_ADMIN_USER)]
  ```

### 3c. No global middleware

`v2/src/backend/app.py` adds **only** `CORSMiddleware`
(`v2/src/backend/app.py:257-263`). There is no auth middleware. Routers are included
plainly (`v2/src/backend/app.py:265-270`):

```python
app.include_router(health.router)
app.include_router(conversation.router)
app.include_router(history.router)
app.include_router(speech.router)
app.include_router(admin.router)
app.include_router(files.router)
```

Auth is enforced **per route** through the `Depends(...)` parameters (`UserIdDep` /
`AdminUserIdDep`), never globally. `GET /api/history/status`
(`v2/src/backend/routers/history.py:52-54`) and `GET /api/admin/config`'s siblings show
the pattern — `history_status` takes only `SettingsDep` and is therefore **unauthenticated**,
while the per-conversation routes take `UserIdDep`.

---

## Q4 — user_id GUID validation

**There is no GUID/UUID format validation of `user_id` anywhere in the backend.**

- The only well-formedness check is `_is_valid_principal_id`
  (`v2/src/backend/dependencies.py:329-344`), which uses a broad character allowlist,
  **not** a GUID pattern (`v2/src/backend/dependencies.py:326`):

  ```python
  _PRINCIPAL_ID_PATTERN = re.compile(r"[A-Za-z0-9._@-]{1,128}")

  def _is_valid_principal_id(value: str) -> bool:
      return _PRINCIPAL_ID_PATTERN.fullmatch(value) is not None
  ```

  Its own docstring says it is "Defensive well-formedness only … rejects
  obviously-garbage values before the id is used as a database partition key; it does
  not assert that the caller is who the id claims to be." It admits Entra object ids, the
  all-zeros default id, and the `local-dev` fallback — a GUID passes, but so does
  `user-oid-123`.

- `uuid` **is** used in the backend, but **only for conversation ids, never user_id**:
  - `v2/src/backend/core/providers/databases/postgres.py:397` —
    `uuid.UUID(conversation_id)` validates/normalizes the **conversation** id (schema
    keys conversations by UUID). `user_id` is stored as `TEXT NOT NULL`
    (`postgres.py:79`) and filtered as a plain string (`WHERE user_id = $1`,
    `postgres.py:384`).
  - `v2/src/backend/core/providers/databases/cosmosdb.py` uses `user_id` verbatim as the
    Cosmos `/userId` partition key (`cosmosdb.py:207`, `partition_key=user_id`) — no GUID
    coercion.
  - `v2/src/backend/models/admin.py:8` and `services/ingestion.py:36` import `uuid4` for
    minting audit/ingestion ids — unrelated to user_id.

- GUID validation **does** exist, but on the **frontend** only:
  `v2/src/frontend/src/api/auth.tsx` defines
  `DEFAULT_USER_ID = "00000000-0000-0000-0000-000000000000"` and the bundled `is-uuid`
  regex (`v2/src/frontend/dist/assets/index-*.js`) validates the value before forwarding
  it. The backend accepts whatever string arrives, subject only to the broad allowlist.

Conclusion: the user's desired behavior ("valid GUID") is **not** implemented on the
backend today; the backend's `user_id` check is a broad-charset presence check, and the
admin routes layer a full Easy Auth role gate on top.

---

## Q5 — How user_id is consumed downstream

The extracted `user_id` is the **tenant partition key** for all chat history. A
replacement must still yield a `str` used as that key.

- Chat endpoint `POST /api/conversation` (`v2/src/backend/routers/conversation.py`):
  - Signature: `user_id: UserIdDep` (`conversation.py:73`).
  - Passed to `persisting_sse_stream(..., user_id=user_id, ...)` (`conversation.py:170`)
    in SSE mode.
  - Passed to `persist_turn(db, user_id=user_id, ...)` (`conversation.py:187`) in
    buffered mode.
  - Also logged in the persist-failure `extra` (`conversation.py:198`).

- History router (`v2/src/backend/routers/history.py`), every tenant-scoped route takes
  `user_id: UserIdDep`:
  - `db.list_conversations(user_id)` (`history.py:59`).
  - `db.create_conversation(user_id=user_id, title=...)` (`history.py:73`).
  - `db.get_conversation(conversation_id, user_id)` + `db.list_messages(...)`
    (`history.py:~82-90`).
  - `db.rename_conversation(...)`, delete, append-message, feedback routes similarly.

- Database layer (`BaseDatabaseClient` implementations):
  - `v2/src/backend/core/providers/databases/base.py:52` — the interface methods take
    `user_id: str`.
  - Cosmos: `/userId` is the partition key; every read/query is single-partition
    (`partition_key=user_id`), e.g. `cosmosdb.py:183`, `:203-207`, `:217`.
  - Postgres: `conversations.user_id TEXT NOT NULL` + index
    `(user_id, updated_at DESC)` (`postgres.py:79-85`); queries filter `WHERE user_id =
    $1` (`postgres.py:384-386`).

Implication for any fix: whatever replaces `get_user_id`/`AdminUserIdDep` must still
return a non-empty `str` to serve as the partition/tenant key, or history isolation and
persistence break.

---

## Q6 — Settings / config flags gating the 401

File: `v2/src/backend/core/settings.py` (`AppSettings`, env prefix `AZURE_`).

Two flags govern the auth posture; both are read by the two extractors above:

- `environment: Environment = Environment.LOCAL` (`v2/src/backend/core/settings.py:527`
  region; the field default line reads `environment: Environment = Environment.LOCAL`).
  - Default: `LOCAL`. Production sets `AZURE_ENVIRONMENT=production` via
    `v2/infra/main.bicep`.
  - Docstring (`settings.py:~516-529`): "`local` is the default so a clean checkout / dev
    run boots without surprises … The admin auth wall is governed separately by
    `require_admin_auth` below, not by this field."

- `require_admin_auth: bool = False` (`v2/src/backend/core/settings.py:~536`).
  - Default: **`False`** (env var `AZURE_REQUIRE_ADMIN_AUTH`).
  - Docstring (`settings.py:530-540`): "`False` (default) leaves admin routes open …
    matching the MACAE-faithful open posture. Set `AZURE_REQUIRE_ADMIN_AUTH=true` to
    require Easy Auth admin-role claims on admin routes; the `requires_role` gate then
    fails closed in any non-`local` environment. A present claims blob is always
    role-checked regardless of this toggle."

How they gate the 401 (both extractors compute the same predicate):

- `allow_open_admin` / `allow_open_auth` = `environment is LOCAL OR not require_admin_auth`.
- 401 fires **only** when that predicate is False → `environment == production` **AND**
  `require_admin_auth == True` **AND** the required header is absent/invalid.

Therefore, with the **shipped code defaults** (`environment=LOCAL`,
`require_admin_auth=False`), the admin routes are open and return `"local-dev"`. The
BUG-0090 401 requires a deployment that set **both** `AZURE_ENVIRONMENT=production` and
`AZURE_REQUIRE_ADMIN_AUTH=true` (the posture in place when the bug was recorded on
2026-06-25). No other flag gates the behavior; there is no `enable_auth` / `disable_auth`
/ `anonymous` flag.

Tests confirm the default: `v2/tests/backend/core/test_settings.py:130`
`test_require_admin_auth_defaults_to_false`.

---

## Q7 — Tests touching the admin router + auth dependency

Any change to the admin auth gate or `user_id` extraction will need edits here.

### `v2/tests/backend/test_dependencies.py` (the auth-gate contract)

Imports `get_user_id` and `requires_role` (`test_dependencies.py:21-22`). Header
constants `_PRINCIPAL_ID = "x-ms-client-principal-id"`, `_PRINCIPAL =
"x-ms-client-principal"` (`:149-150`). The `_settings(...)` helper defaults
`require_admin_auth=True` (`:171`) so the fail-closed tests stay meaningful.

Relevant `requires_role` assertions:

- `test_requires_role_returns_user_id_when_role_present` — 200-path, returns oid.
- `test_requires_role_accepts_full_uri_role_claim` — both role `typ` shapes.
- `test_requires_role_raises_403_when_role_absent` — **403** when authenticated, no role.
- `test_requires_role_raises_401_when_principal_id_missing_in_production` — **401**.
- `test_requires_role_raises_401_when_claims_header_missing_in_production` — **401**
  (this is the BUG-0090 shape: id present, claims blob absent, prod + wall on).
- `test_requires_role_raises_401_on_malformed_base64` / `..._on_malformed_json` — **401**.
- `test_requires_role_falls_back_to_local_dev_when_no_headers_in_local` — returns
  `"local-dev"`.
- `test_requires_role_falls_back_to_local_dev_when_id_present_no_claims_in_local` —
  local bypass keys on absent **claims**, not absent headers; uses the all-zeros id.
- `test_requires_role_in_local_still_validates_when_headers_present` — **403** even in
  local when a `reader` claims blob is forged.
- `test_requires_role_open_admin_returns_user_when_wall_off_in_prod` — with
  `require_admin_auth=False` in prod, missing claims returns `"local-dev"` (no 401).
- `test_requires_role_wall_on_raises_401_without_claims_in_prod` — the fail-closed case.
- The file also has a `get_user_id` block (grep shows `get_user_id` imported at `:21`)
  asserting the lighter chat/history extractor's 401 + local-dev behavior.

### `v2/tests/backend/test_admin.py` (router end-to-end)

- Imports `REQUIRE_ADMIN_USER` (`test_admin.py:42`) and overrides it in the app factory:
  `app.dependency_overrides[REQUIRE_ADMIN_USER] = lambda: "u-1"`
  (`test_admin.py:191`) — i.e. the router tests **bypass** the real gate and pin a fake
  admin user, so they assert status/config **payload** behavior, not the 401 path.
- `_settings(...)` helper defaults `require_admin_auth=True` (`test_admin.py:75`,
  `:102`).
- Status-shape assertions: `test_status_returns_expected_field_set` (`:251`, asserts
  200), `test_status_extracts_foundry_host_only_not_path`,
  `test_status_returns_empty_host_when_endpoint_unset`, `test_status_search_enabled_flag`,
  `test_status_app_insights_enabled_flag`,
  `test_status_maps_orchestrator_db_index_environment`,
  `test_status_reflects_persisted_orchestrator_override`, and a
  `test_status_does_not_leak_sensitive_settings` leak guard (referenced by the router
  docstring). A comment at `test_admin.py:223-226` notes the real 401-through-the-router
  smoke path is covered indirectly and the unit contract lives in `test_dependencies.py`.

### Other tests referencing the headers

- `v2/tests/backend/test_app_exception_handlers.py:84` sends
  `headers={"x-ms-client-principal-id": "user-42"}` (exercises `get_user_id` via a real
  route).
- `v2/tests/backend/test_conversation.py` and `v2/tests/integration/test_admin_live.py`
  reference the auth surface (`test_admin_live.py` is the live/cloud smoke).

Tests that would need to change for a "user_id header present + valid GUID, nothing more"
design:

1. `test_dependencies.py` — all `test_requires_role_*` cases (the whole role-gate
   contract) plus the `get_user_id` cases (if the presence/GUID rule replaces the current
   allowlist + open-posture logic).
2. `test_admin.py` — the `REQUIRE_ADMIN_USER` override (`:191`) and any test asserting the
   admin routes require the `admin` role; the payload-shape tests likely survive
   unchanged if the dependency still returns a `str`.
3. `test_settings.py:130` — if `require_admin_auth` is removed/repurposed.

---

## Should `/api/admin/status` exist?

Yes — it is an intentional, documented Stable Core endpoint. Its module docstring
(`v2/src/backend/routers/admin.py:1-46`) and the tests
(`v2/tests/backend/test_admin.py`) treat it as the sanitized runtime-status snapshot
(orchestrator key, db type, index store, environment, deployment names, feature flags,
CORS list, version) with a leak guard ensuring secrets never surface. The endpoint is
Phase 5, tasks #35a/#39. The open design question is **not** whether the route exists but
**what gate it carries**: today it uses `AdminUserIdDep` (full Easy Auth role gate). A
"only check user_id present + valid GUID" design would swap that for a lighter dependency
(the existing `UserIdDep`, or a new GUID-only presence check) — a per-route dependency
change, no route removal required.

---

## Evidence index (file:line)

- `v2/docs/bugs.md:148-150` — BUG-0089 / BUG-0090 / BUG-0091 registry rows (no BUG-0090
  Details subsection exists).
- `v2/src/backend/routers/admin.py:103` — `APIRouter(prefix="/api/admin")` (no
  router-wide `dependencies=`).
- `v2/src/backend/routers/admin.py:114-118` — `GET /status` + `_user: AdminUserIdDep`.
- `v2/src/backend/routers/admin.py:58-66` — DI imports incl. `AdminUserIdDep`.
- `v2/src/backend/dependencies.py:319-320` — `_PRINCIPAL_ID_HEADER`, `_PRINCIPAL_HEADER`.
- `v2/src/backend/dependencies.py:326` — `_PRINCIPAL_ID_PATTERN` (broad allowlist, not GUID).
- `v2/src/backend/dependencies.py:329-344` — `_is_valid_principal_id`.
- `v2/src/backend/dependencies.py:346-384` — `get_user_id` (chat/history extractor; 401 at
  :370-373 malformed, :379-383 missing).
- `v2/src/backend/dependencies.py:387` — `UserIdDep`.
- `v2/src/backend/dependencies.py:389-406` — `_decode_easy_auth_principal`.
- `v2/src/backend/dependencies.py:408-431` — `_extract_roles`.
- `v2/src/backend/dependencies.py:433-510` — `requires_role` / `_checker`.
- `v2/src/backend/dependencies.py:458-461` — `allow_open_admin` predicate.
- `v2/src/backend/dependencies.py:471-478` — **the BUG-0090 401** ("Missing client
  principal claims; Easy Auth claims header required.").
- `v2/src/backend/dependencies.py:482` / `:489` / `:501` — 401 malformed / 403 no-role /
  401 no-id.
- `v2/src/backend/dependencies.py:513-514` — `REQUIRE_ADMIN_USER`, `AdminUserIdDep`.
- `v2/src/backend/core/settings.py:527` — `environment: Environment = Environment.LOCAL`.
- `v2/src/backend/core/settings.py:~536` — `require_admin_auth: bool = False`.
- `v2/src/backend/app.py:257-270` — CORS-only middleware; plain `include_router` calls.
- `v2/src/backend/routers/history.py:40` — imports `UserIdDep`; `:52-54` unauthenticated
  `/status`; per-conversation routes take `UserIdDep`.
- `v2/src/backend/routers/conversation.py:73,170,187,198` — `user_id` consumption.
- `v2/src/backend/core/providers/databases/cosmosdb.py:207` — `partition_key=user_id`.
- `v2/src/backend/core/providers/databases/postgres.py:79-85,384-386,397` — `user_id
  TEXT` + `WHERE user_id = $1`; `uuid.UUID(conversation_id)` (conversation, not user).
- `v2/src/frontend/src/api/auth.tsx:33,40` — `x-ms-client-principal-id`,
  `DEFAULT_USER_ID = "00000000-..."`; `v2/src/frontend/src/api/admin.tsx:120,147,175,198`
  — admin client forwards only `userIdHeaders()` (the `-id` header), never the claims blob.
- `v2/tests/backend/test_dependencies.py:126-340` — full `requires_role` + `get_user_id`
  contract.
- `v2/tests/backend/test_admin.py:42,191,251+` — router tests, `REQUIRE_ADMIN_USER`
  override.
- `v2/tests/backend/core/test_settings.py:130` —
  `test_require_admin_auth_defaults_to_false`.

---

## Recommended next research (not done this session)

- [ ] Read `v2/src/backend/frontend_app.py` (referenced by BUG-0090 as serving SPA +
  `/config`, not proxying `/api/*`) to confirm the split-host topology and whether option
  (B) reverse-proxy is feasible.
- [ ] Read the `v2/src/frontend/src/api/auth.tsx` `userIdHeaders()` / `getUserInfo()`
  seam in full to confirm exactly which headers the SPA sends to `/api/admin/*` (only the
  `-id` header vs the claims blob).
- [ ] Inspect `v2/infra/main.bicep` to confirm the backend Container App has no
  `authConfig` and whether `AZURE_REQUIRE_ADMIN_AUTH` is wired on the backend env-vars.
- [ ] Confirm exact `settings.py` field line numbers for `require_admin_auth` (read the
  516-560 window; the field is just below the `environment` field comment block).

## Clarifying questions for the user

1. **Desired final posture on `/api/admin/*`:** should the admin routes use the *same*
   lighter check as chat/history (`UserIdDep` → header present + broad allowlist), or a
   *new* strict "present AND valid GUID" check that also rejects the current `local-dev`
   and `user-oid-*` style ids? (The current `local-dev` fallback is not a GUID and would
   fail a strict GUID rule — that would break local dev and the open-posture default.)
2. **Keep or drop `require_admin_auth` / Easy Auth role gate entirely?** The user's stated
   intent ("no Easy Auth, no principal parsing, no 401 gating") implies removing
   `requires_role`/`AdminUserIdDep` and the `x-ms-client-principal` claims parsing
   altogether. Confirm that admin write routes (PATCH config, upload, delete, reprocess)
   should also drop role enforcement, or whether only `/status` (read-only) should relax.
3. **GUID validity definition:** accept the all-zeros `DEFAULT_USER_ID` guest guid
   (`00000000-0000-0000-0000-000000000000`) as valid, or reject it? The frontend sends it
   as the default when no principal resolves.
