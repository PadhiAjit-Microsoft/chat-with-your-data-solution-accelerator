<!-- markdownlint-disable-file -->
# Implementation Details: BUG-0090 — Admin 401 & user_id header handling

## Context Reference

Sources:
* .copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md (primary — selected approach, alternatives, security tradeoff)
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-backend-auth-wiring-research.md (backend wiring map, file:line index)
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-macae-user-id-pattern-research.md (MACAE contract to mirror)
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-frontend-user-id-research.md (frontend already compliant)
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-bicep-env-and-environment-usage-research.md (Bicep env wiring + `settings.environment` consumer audit)

Selected decision (PD-01): on a present-but-invalid GUID, `get_user_id` falls back to the default GUID (never raises) — MACAE-faithful. Hard-400 variant rejected as default (see log DD-01).
Selected decision (PD-02): keep the `environment` field + `Environment` enum (minimal-change path) because `v2/src/backend/routers/admin.py:138` still feeds `AdminStatus.environment`. Only the two auth reads are removed.

## Implementation Phase 1: Minimal `get_user_id` (present + valid GUID → use; else default GUID)

<!-- parallelizable: false -->

### Step 1.1: Rewrite `get_user_id` + add `_is_valid_guid` / `_DEFAULT_USER_ID`; remove the exclusive `_is_valid_principal_id` allowlist

Rewrite `get_user_id` in `v2/src/backend/dependencies.py` so it takes only `Request` (drop `SettingsDep`), reads `x-ms-client-principal-id`, returns it iff it is a valid GUID, else returns the anonymous default GUID `00000000-0000-0000-0000-000000000000`. It must NEVER raise. Add a private `_is_valid_guid(value: str) -> bool` (wraps `uuid.UUID(value)` in try/except `ValueError`) and a `_DEFAULT_USER_ID` module constant. Remove `_is_valid_principal_id` and `_PRINCIPAL_ID_PATTERN` (used ONLY by the old `get_user_id`). Do NOT remove `_LOCAL_DEV_USER` yet — `requires_role._checker` still references it until Phase 3. Keep `_PRINCIPAL_ID_HEADER`.

Target current code (from research):
* `get_user_id` — v2/src/backend/dependencies.py:346-384 (signature `def get_user_id(request: Request, settings: SettingsDep) -> str`; 401 raises at :370 malformed, :379 missing).
* `_is_valid_principal_id` — v2/src/backend/dependencies.py:329-344; `_PRINCIPAL_ID_PATTERN` — :326.
* `UserIdDep = Annotated[str, Depends(get_user_id)]` — :387 (unchanged alias; consumers in history.py/conversation.py unaffected).
* `import uuid` — confirm/add at the top import block (Hard Rule #17 all-imports-at-top).

New shape (from research Complete Example):
```python
_DEFAULT_USER_ID = "00000000-0000-0000-0000-000000000000"  # anon default GUID (matches FE + MACAE)

def _is_valid_guid(value: str) -> bool:
    try:
        uuid.UUID(value)
        return True
    except ValueError:
        return False

def get_user_id(request: Request) -> str:
    raw = request.headers.get(_PRINCIPAL_ID_HEADER, "").strip()
    if raw and _is_valid_guid(raw):
        return raw
    return _DEFAULT_USER_ID
```

Files:
* v2/src/backend/dependencies.py - rewrite `get_user_id`; add `_is_valid_guid` + `_DEFAULT_USER_ID`; delete `_is_valid_principal_id` + `_PRINCIPAL_ID_PATTERN`.

Discrepancy references:
* Implements research §"Selected approach" and the Complete Example.
* Addresses DD-01 (PD-01 fallback-to-default chosen).

Success criteria:
* `get_user_id(request)` returns the header value when it is a valid GUID.
* Returns `00000000-0000-0000-0000-000000000000` when the header is missing, blank, or not a valid GUID.
* Never raises `HTTPException`; no `settings` parameter; no `environment`/`require_admin_auth` reference.
* `_is_valid_principal_id` / `_PRINCIPAL_ID_PATTERN` are gone; no remaining references (grep clean).

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-backend-auth-wiring-research.md (Q3a, Q4) - current `get_user_id` + allowlist.
* .copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md ("Complete Examples").

Dependencies:
* None (foundation unit).

### Step 1.2: Rewrite EVERY `get_user_id` contract test for the new GUID rule (four files)

The `get_user_id` contract is asserted in FOUR backend test files (validator DR-04/DD-03), all of which break under the new signature (drops the `settings` positional arg), the removed `"local-dev"` / 401 fallback, and the removed non-GUID echo. Update all of them to the new contract: valid-GUID header → returned verbatim; missing/blank/non-GUID → `00000000-0000-0000-0000-000000000000`; never raises; `get_user_id(request)` called with NO `settings` argument. Audit every non-GUID test principal (`user-42`, `user-oid-123/456/789`, `local-dev`, the `u-1` override) and either (a) replace the input header with a real GUID so the "echoes the caller's id" intent is preserved, or (b) change the expected output to the default GUID when the test's point is the anonymous fallback.

Test sites to change (verified by grep):
* v2/tests/backend/test_history.py:106-127 - the `test_get_user_id_*` unit suite (`test_get_user_id_reads_easy_auth_header`, `test_get_user_id_falls_back_to_local_dev_when_header_missing`, `test_get_user_id_raises_401_in_production_when_header_missing`); all call `get_user_id(Request(scope), settings)` (2 positional args) and assert `== "local-dev"` / the 401. Rewrite to the GUID contract; rename the `_falls_back_to_local_dev_*` / `_raises_401_*` cases to `_falls_back_to_default_guid_*`. Also the `app.dependency_overrides[get_user_id] = lambda: "u-1"` at test_history.py:94 - change `"u-1"` to a valid GUID for realism (optional but preferred).
* v2/tests/backend/test_dependencies.py - the second `get_user_id` block ("get_user_id -- Easy Auth principal extraction", ~line 570+): rewrite to the GUID contract; drop the `settings` arg. (Leave the `requires_role` suite alone here - deleted in Phase 3.)
* v2/tests/backend/test_conversation.py:100,1024 - the `local-dev` fallback narrative + assertion → default GUID; test_conversation.py:1119-1120 - the `user-42` echo → use a valid GUID header so it echoes.
* v2/tests/backend/test_app_exception_handlers.py:84,95 - `headers={"x-ms-client-principal-id": "user-42"}` + `assert record.user_id == "user-42"` → change `user-42` to a valid GUID so the echo assertion holds.

Files:
* v2/tests/backend/test_history.py - rewrite the `test_get_user_id_*` suite.
* v2/tests/backend/test_dependencies.py - rewrite the `get_user_id` block only.
* v2/tests/backend/test_conversation.py - default-GUID fallback + GUID echo.
* v2/tests/backend/test_app_exception_handlers.py - GUID principal.

Success criteria:
* No test calls `get_user_id(...)` with a `settings` argument (grep clean).
* No test asserts `== "local-dev"` for `get_user_id` output or echoes a non-GUID principal (`user-42`, `user-oid-*`).
* New/renamed cases assert: valid GUID returned verbatim; missing/blank/non-GUID → `00000000-0000-0000-0000-000000000000`.
* `python -m pytest v2/tests/backend/test_history.py v2/tests/backend/test_conversation.py v2/tests/backend/test_app_exception_handlers.py -q` passes (the `get_user_id` block of test_dependencies.py also passes; its `requires_role` block still passes until Phase 3).

Context references:
* .copilot-tracking/plans/logs/2026-07-02/bug-0090-admin-user-id-log.md (DR-04, DD-03) - the four-file test inventory.
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-backend-auth-wiring-research.md (Q7).

Dependencies:
* Step 1.1 completion.

## Implementation Phase 2: Point admin routes at `UserIdDep` (remove the role gate from routes)

<!-- parallelizable: false -->

### Step 2.1: Swap `AdminUserIdDep` → `UserIdDep` on all `/api/admin/*` routes

In `v2/src/backend/routers/admin.py`, replace every `AdminUserIdDep` route parameter with `UserIdDep` (status, config, config/effective, and the PATCH/upload/reprocess/ingest/delete write routes). Update the import (`AdminUserIdDep` → `UserIdDep` from `backend.dependencies`; `UserIdDep` already exists). Keep `SettingsDep`/`RuntimeOverridesDep`/etc. Rewrite the FULL module docstring (validator DD-04: the role-gate narrative begins at line ~24 with the `requires_role` factory mention and carries a `#39` task token, ABOVE the originally-cited 29-46 span) so it describes the new contract (reads `x-ms-client-principal-id`, valid-GUID-or-default, no role gate) — present-tense, and remove the `requires_role` / `local-dev` / `#39` narrative (Hard Rule #16; leaving `requires_role` in the docstring also fails Phase 3.1's grep-clean criterion).

Target current code:
* `router = APIRouter(prefix="/api/admin", ...)` — v2/src/backend/routers/admin.py:103 (no router-wide `dependencies=`).
* `GET /status` `_user: AdminUserIdDep` — v2/src/backend/routers/admin.py:114-118.
* DI import incl. `AdminUserIdDep` — v2/src/backend/routers/admin.py:58-66.
* `environment=settings.environment` in the `AdminStatus(...)` build — v2/src/backend/routers/admin.py:138 (UNCHANGED — keep).

Files:
* v2/src/backend/routers/admin.py - replace `AdminUserIdDep` with `UserIdDep` on all routes; update import + docstring.

Discrepancy references:
* Implements research §"Selected approach" (uniform `UserIdDep`).
* Rejects Alt C (partial relax) — all admin routes swapped, not just `/status`.

Success criteria:
* No `AdminUserIdDep` reference remains in `admin.py` (grep clean).
* All `/api/admin/*` routes depend on `UserIdDep`.
* `AdminStatus.environment` still populated from `settings.environment` (line 138 untouched).
* Module docstring reflects the header-GUID contract with no role/Easy-Auth narrative.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-backend-auth-wiring-research.md (Q2, "Should /api/admin/status exist?").

Dependencies:
* Step 1.1 (UserIdDep signature is now settings-free).

### Step 2.2: Update `test_admin.py` to override `get_user_id` / `UserIdDep` instead of `REQUIRE_ADMIN_USER`

In `v2/tests/backend/test_admin.py`, replace the `app.dependency_overrides[REQUIRE_ADMIN_USER] = lambda: "u-1"` override (test_admin.py:191) with an override of `get_user_id` returning a fixed GUID. Drop the `REQUIRE_ADMIN_USER` import (test_admin.py:42). Also update the module docstring (test_admin.py:6-8) which references `#39 RBAC-narrowed REQUIRE_ADMIN_USER` and `test_dependencies.py::test_requires_role_*` — both deleted; rewrite it to the new contract (test files are Hard Rule #16-exempt, but the stale symbol references must go for accuracy + grep-clean). The `require_admin_auth=True` in the `_settings(...)` helper (test_admin.py:75, :102) is removed in Phase 3.2 — for this phase, leave it (the router no longer reads it). Keep all status-shape/leak-guard assertions (they still pass — dependency still returns a `str`).

Files:
* v2/tests/backend/test_admin.py - swap the dependency override; drop `REQUIRE_ADMIN_USER` import; refresh the module docstring.

Success criteria:
* `test_admin.py` overrides `get_user_id`, not `REQUIRE_ADMIN_USER`.
* `python -m pytest v2/tests/backend/test_admin.py -q` passes (status-shape + leak-guard tests green).
* `GET /api/admin/status` returns 200 in the test client with a GUID user override and no claims blob.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-backend-auth-wiring-research.md (Q7 - test_admin.py:42,191,251+).

Dependencies:
* Step 2.1 completion.

## Implementation Phase 3: Delete the dead Easy Auth role-gate cluster + `require_admin_auth` setting

<!-- parallelizable: false -->

### Step 3.1: Remove the role-gate cluster from `dependencies.py` and the `require_admin_auth` field from `settings.py`

After Phase 2, the role gate has zero production callers. Verify with a tree grep, then delete the whole cluster from `v2/src/backend/dependencies.py`: `requires_role` + its `_checker`, `_decode_easy_auth_principal`, `_extract_roles`, `REQUIRE_ADMIN_USER`, `AdminUserIdDep`, `_PRINCIPAL_HEADER`, the role-typ constants (`_ROLE_TYP_SHORT` / `_ROLE_TYP_FULL`), and `_LOCAL_DEV_USER` (now unused). Remove any imports left dangling (e.g. `base64`, `json`, `Environment` if unused elsewhere in the file — but `Environment` is still imported at dependencies.py:39; confirm it is unused after this and drop the import if so). In `v2/src/backend/core/settings.py`, delete the `require_admin_auth: bool = False` field (settings.py:543) and its comment (settings.py:535-542). KEEP `environment: Environment = Environment.LOCAL` (settings.py:533) and the `Environment` enum (settings.py:41-52) — still used by `admin.py:138` — BUT scrub the surviving `require_admin_auth` cross-reference in the KEPT `environment` field comment (settings.py:527, validator DR-07: it currently says the admin wall is "governed separately by `require_admin_auth`") so the Step 3.1 grep-clean criterion holds; rewrite that comment to describe `environment` as a status-report value only. Additionally, rewrite the `RuntimeConfig.updated_by` field docstring in `v2/src/backend/core/types.py:320` (validator DR-05), which currently reads "...from the `REQUIRE_ADMIN_USER` dep ... built via `backend.dependencies.requires_role(\"admin\")`" — both deleted symbols; change it to "carries the admin caller's user id (from `get_user_id` / `UserIdDep` in `backend.dependencies`)". Finally, scrub the stale `local-dev` / 401 narrative left in three Stable Core docstrings (validator DR-08, Hard Rule #16): the `Environment` enum docstring (settings.py:41-52), and the module/route docstrings in `v2/src/backend/routers/history.py` and `v2/src/backend/routers/conversation.py` that describe the old `get_user_id` fallback — update them to the new default-GUID contract. Without all of these, Step 3.1's grep-clean + no-stale-narrative criteria fail.

Target current code:
* `requires_role` / `_checker` — v2/src/backend/dependencies.py:433-510.
* `_decode_easy_auth_principal` — :389-406; `_extract_roles` — :408-431.
* `REQUIRE_ADMIN_USER`, `AdminUserIdDep` — :513-514.
* `_PRINCIPAL_HEADER` — :315 (with `_PRINCIPAL_ID_HEADER` at :314); role-typ constants near the header constants. (Deletion is grep-driven, not line-number-driven — validator DD-06.)
* `_LOCAL_DEV_USER` — used by both extractors pre-refactor; now dead.
* `Environment` import — dependencies.py:39.
* `require_admin_auth` field — v2/src/backend/core/settings.py:543.
* `RuntimeConfig.updated_by` docstring — v2/src/backend/core/types.py:320 (rewrite; references deleted symbols).

Grep-before-delete (must show ONLY test + self references):
* `requires_role`, `REQUIRE_ADMIN_USER`, `AdminUserIdDep`, `require_admin_auth`, `_decode_easy_auth_principal`, `_extract_roles`, `_LOCAL_DEV_USER`, `_PRINCIPAL_HEADER` across `v2/src/**`.

Files:
* v2/src/backend/dependencies.py - delete the role-gate cluster + now-dead constants/imports.
* v2/src/backend/core/settings.py - delete `require_admin_auth` field + comment; scrub the `require_admin_auth` reference + stale narrative from the kept `environment` field comment + `Environment` enum docstring.
* v2/src/backend/core/types.py - rewrite the `RuntimeConfig.updated_by` docstring (drop the deleted `REQUIRE_ADMIN_USER` / `requires_role` references).
* v2/src/backend/routers/history.py - scrub stale `local-dev`/401 `get_user_id` narrative from docstrings (doc-only).
* v2/src/backend/routers/conversation.py - scrub stale `local-dev`/401 `get_user_id` narrative from docstrings (doc-only).

Discrepancy references:
* Implements research §"Preferred Approach" step 3 (cleanup / reduce code debt).
* Addresses DR-01 (`require_admin_auth` fully removed), DR-05 (types.py docstring), DR-07 (environment-field comment), DR-08 (stale docstrings).

Success criteria:
* No `requires_role` / `REQUIRE_ADMIN_USER` / `AdminUserIdDep` / `_decode_easy_auth_principal` / `_extract_roles` / `_LOCAL_DEV_USER` / `_PRINCIPAL_HEADER` / role-typ constants remain in `v2/src/**` (grep clean — INCLUDING docstrings, so types.py:320 must be rewritten).
* No `require_admin_auth` remains in `v2/src/**` (grep clean — INCLUDING the kept `environment` field comment at settings.py:527).
* No stale `local-dev` / "Easy Auth claims" / 401-fallback narrative remains in `settings.py` (`Environment` docstring), `history.py`, or `conversation.py` (Hard Rule #16).
* `environment` field + `Environment` enum retained; `admin.py:138` still compiles.
* No dead imports left in `dependencies.py` (`base64`/`json`/`Environment` removed if unused).
* `get_errors` on `dependencies.py` + `settings.py` + `types.py` + `history.py` + `conversation.py` clean.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-backend-auth-wiring-research.md (Q3b, Q6, evidence index).
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-bicep-env-and-environment-usage-research.md (Gap 2 - `environment` kept for admin.py:138; `require_admin_auth` fully deletable).

Dependencies:
* Phase 2 completion (routes no longer reference the gate).

### Step 3.2: Delete the role-gate + `require_admin_auth` tests

Remove the now-orphaned tests: the entire `test_requires_role_*` suite and any `_decode`/`_extract_roles` unit tests in `v2/tests/backend/test_dependencies.py` (plus the `requires_role`/`REQUIRE_ADMIN_USER` imports at :21-22); `test_require_admin_auth_defaults_to_false` and `test_require_admin_auth_env_override_enables_wall` in `v2/tests/backend/core/test_settings.py`; and the `require_admin_auth=...` argument (+ its `_settings` default at :171) from `v2/tests/backend/test_admin.py` (test_admin.py:75, :102). Also update the Hard Rule #15 boundary allow-list gate `v2/tests/shared/test_no_anonymous_dict_returns.py:37` (validator DR-06), which exempts `backend.dependencies._decode_easy_auth_principal` — remove that exemption entry + its docstring bullet, since the symbol is deleted in Step 3.1. Verify no other test imports the deleted symbols.

Files:
* v2/tests/backend/test_dependencies.py - delete `test_requires_role_*` + claims-helper tests + their imports.
* v2/tests/backend/core/test_settings.py - delete the two `require_admin_auth` tests.
* v2/tests/backend/test_admin.py - drop `require_admin_auth` from `_settings(...)`.
* v2/tests/shared/test_no_anonymous_dict_returns.py - remove the `_decode_easy_auth_principal` exemption entry + docstring bullet.

Success criteria:
* No test references `requires_role` / `REQUIRE_ADMIN_USER` / `require_admin_auth` / `_decode_easy_auth_principal` (grep clean under `v2/tests/**`).
* The Hard Rule #15 gate (`test_no_anonymous_dict_returns.py`) passes with no stale exemption for a deleted symbol.
* `python -m pytest v2/tests/backend v2/tests/shared -q` passes.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-backend-auth-wiring-research.md (Q7).

Dependencies:
* Step 3.1 completion.

## Implementation Phase 4: Bicep — remove `AZURE_REQUIRE_ADMIN_AUTH`, refresh `AZURE_ENVIRONMENT` comment

<!-- parallelizable: true -->

### Step 4.1: Delete the `AZURE_REQUIRE_ADMIN_AUTH` backend env entry; update the stale `AZURE_ENVIRONMENT` comment

In `v2/infra/main.bicep` backend container-app env block: delete the `{ name: 'AZURE_REQUIRE_ADMIN_AUTH', value: 'false' }` entry (main.bicep:1813) and its comment (main.bicep:1806-1812). KEEP `{ name: 'AZURE_ENVIRONMENT', value: 'production' }` (main.bicep:1805) — it still feeds `AdminStatus.environment` — but rewrite its comment (main.bicep:1797-1805) to an accurate present-tense note: `AZURE_ENVIRONMENT` sets `AppSettings.environment`, surfaced by `GET /api/admin/status`; it no longer governs any auth behavior. Leave the functions `AZURE_ENVIRONMENT` (main.bicep:2160) and its comment as-is (still valid: parity status value). Do not introduce env-specific IDs (Hard Rule #18).

Target current code (exact snippets in the bicep research doc):
* Backend `AZURE_ENVIRONMENT` — v2/infra/main.bicep:1805 (comment 1797-1805).
* Backend `AZURE_REQUIRE_ADMIN_AUTH` — v2/infra/main.bicep:1813 (comment 1806-1812).
* Functions `AZURE_ENVIRONMENT` — v2/infra/main.bicep:2160 (UNCHANGED).

Files:
* v2/infra/main.bicep - remove the backend `AZURE_REQUIRE_ADMIN_AUTH` env entry + comment; rewrite the backend `AZURE_ENVIRONMENT` comment.

Discrepancy references:
* Implements research §"Configuration Examples".
* Addresses DR-02 (dangling env var after field deletion; `extra="ignore"` makes it non-fatal but we remove it for cleanliness).

Success criteria:
* No `AZURE_REQUIRE_ADMIN_AUTH` remains anywhere in `v2/infra/**` (grep clean).
* Backend `AZURE_ENVIRONMENT` retained with an accurate, auth-free comment.
* `az bicep build v2/infra/main.bicep` exits 0 (or the repo's bicep build task passes).

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-bicep-env-and-environment-usage-research.md (Gap 1 - snippets + `extra="ignore"` note).

Dependencies:
* Independent of the Python phases (different files). May run any time; no build coupling.

### Step 4.2: Update infra tests if they assert on the removed env var

Check `v2/tests/infra/test_main_bicep.py` for any assertion referencing `AZURE_REQUIRE_ADMIN_AUTH` and remove/adjust it. If none exists, add a small assertion that `AZURE_REQUIRE_ADMIN_AUTH` is absent from the backend env block and `AZURE_ENVIRONMENT` remains (test-first hygiene, Hard Rule #2).

Files:
* v2/tests/infra/test_main_bicep.py - remove/adjust any `AZURE_REQUIRE_ADMIN_AUTH` assertion; assert its absence + `AZURE_ENVIRONMENT` presence.

Success criteria:
* `python -m pytest v2/tests/infra/test_main_bicep.py -q` passes.
* A test pins `AZURE_REQUIRE_ADMIN_AUTH` absence in the backend env block.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-bicep-env-and-environment-usage-research.md (Gap 1).

Dependencies:
* Step 4.1 completion.

## Implementation Phase 5: Documentation — BUG-0090 correction, worklog, ADR, frontend no-op verification

<!-- parallelizable: false -->

### Step 5.1: Correct the BUG-0090 registry row + append a worklog entry

In `v2/docs/bugs.md`, rewrite the BUG-0090 registry row (bugs.md:149) so the root cause is accurate (the `requires_role("admin")` gate at dependencies.py:472, active only when `environment=production` + `require_admin_auth=true`; the SPA never sends the base64 claims blob) and the fix is the delivered approach (collapse to a single `get_user_id` that validates the `x-ms-client-principal-id` header is a GUID and falls back to the default GUID; delete the role gate + `require_admin_auth`; remove `AZURE_REQUIRE_ADMIN_AUTH` from Bicep; identity enforcement is ingress-level). Set Status → `fixed` and the Fixed date ONLY after Phase 6 verification (leave `open` until then). Use placeholders for any resource names (Hard Rule #18). Append a dated entry to `v2/docs/worklog/2026-07-02.md` summarizing the change and the security tradeoff.

Files:
* v2/docs/bugs.md - correct BUG-0090 root cause + fix; flip status at verification.
* v2/docs/worklog/2026-07-02.md - append the BUG-0090 entry.

Success criteria:
* BUG-0090 row states the real root cause + delivered fix; no stale "wire Easy Auth" recommendation.
* Worklog entry records the change + the forgeable-header / ingress-auth tradeoff.
* Env-ID gate (`v2/tests/**/test_no_env_specific_content.py`) passes on both files.

Context references:
* .copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md ("Corrected BUG-0090 root cause").

Dependencies:
* Phases 1-4 implemented (fix content known); status flip waits on Phase 6.

### Step 5.2: Author (or amend) an ADR for the auth-posture decision

Search `v2/docs/adr/` for an existing ADR covering the admin auth wall / `require_admin_auth` / Easy Auth posture. If one exists, add an Amendment documenting: the backend no longer enforces admin auth in application code; `user_id` arrives as a trusted (client-forgeable) `x-ms-client-principal-id` header validated only as a GUID; real authentication, when enabled, is an ingress/frontend concern (Easy Auth injecting/overwriting the header), matching MACAE. If none exists, author a new sequentially-numbered ADR with the same content, including the security tradeoff and the revert path (re-add the gate + flag). No env-specific IDs.

Files:
* v2/docs/adr/NNNN-*.md - new ADR or amendment documenting the posture + tradeoff + revert path.

Success criteria:
* An ADR (new or amended) records the decision, the forgeable-header tradeoff, the ingress-level enforcement model, and the revert path.
* Markdown lint clean; no env-specific IDs.

Context references:
* .copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md ("Documented security tradeoff").

Dependencies:
* None (documentation).

### Step 5.3: Verify the frontend is already compliant (no code change)

Confirm — without editing — that the frontend already: sends `x-ms-client-principal-id` on every user-facing request (`v2/src/frontend/src/api/auth.tsx:33`), defaults to `DEFAULT_USER_ID = "00000000-0000-0000-0000-000000000000"` when anonymous (auth.tsx:40, a valid GUID), and renders `G` for Guest (components/Header/userIdentity.tsx). Record this as a no-op verification in the worklog. No frontend file is modified.

Files:
* (none modified) - verification only.

Success criteria:
* Verification note in the worklog confirming the frontend already matches the intended behavior; zero frontend edits.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-frontend-user-id-research.md.

Dependencies:
* None.

## Implementation Phase 6: Validation

<!-- parallelizable: false -->

### Step 6.1: Run backend + infra tests, bicep build, env-ID gate

Execute the full relevant validation set:
* Backend: `C:\workstation\Microsoft\github\cwyd-cdb\v2\.venv\Scripts\python.exe -m pytest v2/tests/backend -q`
* Infra: `...python.exe -m pytest v2/tests/infra -q`
* Bicep build: the repo's `az bicep build` on `v2/infra/main.bicep` (exit 0).
* Env-ID gate: `...python.exe -m pytest v2/tests -k no_env_specific -q`
* Shared invariant gates (imports-at-top, no-process-narrative, init-marker) for touched `v2/src` files.

### Step 6.2: Fix minor validation issues

Iterate on lint/type/test failures scoped to the touched files (`pyright --strict` on `v2/src/backend/**`; ruff). Apply straightforward fixes directly.

### Step 6.3: Live deploy + verify (gated on user go-ahead)

This is a structural change (Hard Rule #10) — obtain explicit user go-ahead before deploying. Then, from a terminal already at `v2` cwd: `azd provision` (or `azd deploy backend`), and verify `GET https://<backend-fqdn>/api/admin/status` returns 200 (not 401) with a GUID `x-ms-client-principal-id` header, and 200 with the default GUID. On success, flip BUG-0090 → `fixed` (Step 5.1) and log the verification. Clean up any test artifacts afterward (memory: cleanup-before-next-step).

### Step 6.4: Report blocking issues

Document any failure needing further research; provide next steps rather than large inline fixes.

Files:
* (validation only; no new production files)

Success criteria:
* All backend + infra tests green; bicep build exit 0; env-ID gate + shared gates green.
* `pyright --strict` clean on touched backend files.
* (Post go-ahead) `/api/admin/status` returns 200 for GUID + default-GUID headers in the live deployment.

Dependencies:
* Phases 1-5 completion.

## Dependencies

* Python 3.11+ venv at `v2/.venv` (`v2\.venv\Scripts\python.exe`); pytest.
* Azure CLI / `az bicep` for the bicep build; `azd` (v2 cwd) for the optional live deploy.
* FastAPI + Pydantic v2 (existing).

## Success Criteria

* `/api/admin/status` (and every `/api/admin/*` route) can no longer return the Easy-Auth 401 — the role gate is gone. — Traces to: user request; research "Corrected root cause".
* Backend `get_user_id` is a single dependency: header present + valid GUID → use it; else default GUID; never raises. — Traces to: user request ("only check present + valid GUID"); MACAE pattern.
* The Easy Auth role-gate cluster + `require_admin_auth` setting + `AZURE_REQUIRE_ADMIN_AUTH` Bicep entry are fully removed (no dead code). — Traces to: cleanup-before-next-step memory; research "Preferred Approach".
* `environment` field retained (feeds `AdminStatus.environment`); frontend unchanged (already compliant). — Traces to: bicep-env research Gap 2; frontend research.
* BUG-0090 registry + worklog + ADR corrected/recorded; all gates green. — Traces to: Hard Rule #19.
