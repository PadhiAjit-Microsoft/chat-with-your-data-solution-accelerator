<!-- markdownlint-disable-file -->
# RPI Validation — BUG-0090 Phase 1 (Minimal `get_user_id`)

- **Plan**: `.copilot-tracking/plans/2026-07-02/bug-0090-admin-user-id-plan.instructions.md`
- **Details**: `.copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md` (Phase 1 = Lines 16–96)
- **Changes log**: `.copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md`
- **Research**: `.copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md`
- **Phase**: 1 — Minimal `get_user_id`
- **Validation date**: 2026-07-02
- **Method**: read-only comparison of plan/details/research against the ACTUAL source (`v2/src/backend/dependencies.py`) + four test files + tree-wide grep.

## Status: PASS

All Phase 1 plan items (Step 1.1 + Step 1.2) are implemented in the actual source and tests, and the changes-log claims match the on-disk state. No Critical or Major findings. Two Minor/informational notes are recorded below; neither is in Phase 1 scope.

---

## Requirement-by-requirement verdicts

### 1. `get_user_id` is Request-only, header-driven, GUID-or-default, never raises — PASS

`v2/src/backend/dependencies.py:286-297`:

```python
def get_user_id(request: Request) -> str:
    ...
    raw = request.headers.get(_PRINCIPAL_ID_HEADER, "").strip()
    if raw and _is_valid_guid(raw):
        return raw
    return _DEFAULT_USER_ID
```

- Signature is `def get_user_id(request: Request) -> str` — **no `settings` parameter** (`dependencies.py:286`). Matches details Step 1.1 New shape and research "Complete Examples".
- Reads `_PRINCIPAL_ID_HEADER = "x-ms-client-principal-id"` (`dependencies.py:273`, used at `:294`).
- Returns the header verbatim only when it is a valid GUID (`:295-296`); missing / blank / non-GUID → `_DEFAULT_USER_ID` (`:297`).
- **Never raises** — no `HTTPException`/`raise` in the function body; `_is_valid_guid` swallows `ValueError` (`dependencies.py:277-283`).
- Default id is the anonymous all-zeros GUID `00000000-0000-0000-0000-000000000000` (`dependencies.py:274`), matching the frontend sentinel and MACAE (research §"Complete Examples").

### 2. Helpers present, allowlist gone, imports correct — PASS

- `_is_valid_guid(value: str) -> bool` present, wraps `uuid.UUID(value)` in `try/except ValueError` — `dependencies.py:277-283`.
- `_DEFAULT_USER_ID` module constant present — `dependencies.py:274`.
- `_is_valid_principal_id` and `_PRINCIPAL_ID_PATTERN` are **removed** — tree-wide grep returns **zero** hits under `v2/src/**` and `v2/tests/**` (only immutable worklog history under `v2/docs/worklog/2026-06-15.md` still references the retired symbol, which is correct).
- `import uuid` is at the **module top** — `dependencies.py:19` (Hard Rule #17). Used by `_is_valid_guid`.
- `import re` is **removed**: grep for `import re` and `\bre\.` in `dependencies.py` returns **no matches** — no live `re.` use remains. Matches changes-log claim "`import re` → `import uuid`".

### 3. `UserIdDep` intact; Phase-3 role-gate symbols not a Phase 1 concern — PASS

- `UserIdDep = Annotated[str, Depends(get_user_id)]` present and unchanged — `dependencies.py:300`; also exported in `__all__`.
- The changes log attributes the role-gate cluster deletion (`requires_role`, `AdminUserIdDep`, `REQUIRE_ADMIN_USER`, `_PRINCIPAL_ID_PATTERN`-adjacent role symbols, `_LOCAL_DEV_USER`) exclusively to **Phase 3** ("Removed … [Phase 3]"), and its Phase 1 entries do not claim deleting those symbols. Correct phase attribution — Phase 1 touched only `get_user_id` + the allowlist pair (`_is_valid_principal_id` / `_PRINCIPAL_ID_PATTERN`) that were used **only** by the old `get_user_id`, exactly as details Step 1.1 scopes it.
- The current source has the role-gate symbols gone because all later phases are complete; this reflects Phase 3, not a Phase 1 overreach.

### 4. Test files — GUID contract, no `settings` arg, no `local-dev`, non-GUID principals migrated — PASS

Tree-wide grep confirms **no** `get_user_id(...)` call passes a `settings` argument and **no** test asserts `== "local-dev"` for `get_user_id` output (only worklog docs mention the string).

- `v2/tests/backend/test_history.py`
  - `get_user_id` unit suite rewritten to the GUID contract; all calls pass a single `Request` arg (`:121, :129, :140, :157`).
  - Cases renamed to `_falls_back_to_default_guid_*` (`:126, :134`); valid-GUID echo (`:110-123`); all-zeros default accepted (`:145-159`).
  - `_TEST_USER_ID = "3f2504e0-4f89-41d3-9a0c-0305e82c3301"` (a valid GUID) threaded through the `get_user_id` override (`:94`) and downstream router assertions — matches the changes-log "Additional/Deviating" note.
- `v2/tests/backend/test_dependencies.py`
  - Second `get_user_id` block rewritten to the GUID contract (`:333-345`); each call is `get_user_id(request)` (one arg).
  - Non-GUID input `user-oid-42` is used as **input** with expected output `00000000-0000-0000-0000-000000000000` (`:343-345`) — satisfies details Step 1.2 option (b) "expected default GUID".
- `v2/tests/backend/test_conversation.py`
  - Anonymous-fallback docstring + assertions use the default GUID (`:99-101, :1026, :1099`).
  - Principal-echo test uses a valid GUID header `11111111-1111-1111-1111-111111111111` and asserts it echoes (`:1113, :1120, :1127`) — the former `user-42` echo is migrated.
- `v2/tests/backend/test_app_exception_handlers.py`
  - `user-42` migrated to the valid GUID `22222222-2222-2222-2222-222222222222` for the header + echo assertion (`:75, :85, :96`) — matches details Step 1.2 and the changes-log claim.

### 5. Hard Rules #16 (no process narrative) + #17 (imports at top) — PASS

- New docstrings on `get_user_id` (`dependencies.py:287-293`) and `_is_valid_guid` (`:278`) contain **no** unit IDs, **no** `#39` token, **no** dates, and **no** "local-dev" narrative — present-tense contract description only (Hard Rule #16).
- All imports (incl. `import uuid`) are at the module top; no in-function imports in the Phase 1 code (Hard Rule #17).

---

## Findings (severity-ordered)

### Critical

None.

### Major

None.

### Minor

- **M1 — Stale live docs reference removed symbols (OUT OF PHASE 1 SCOPE).** `v2/docs/mvp_status.md:125,148` and `v2/docs/admin_runtime_config.md:20` still describe the removed `requires_role("admin")` / `AdminUserIdDep` contract as current, and `v2/docs/development_plan.md:115,169` carries the `#39` role-gate narrative. These are **not** Phase 1 artifacts (Phase 1 = `get_user_id` + its four test files); the plan's doc-scrub work lives in Phase 3 (Stable Core docstrings) and Phase 5 (bugs.md/worklog/ADR). Immutable worklogs and ADRs correctly retain the historical references. Recorded as an informational carry-forward for the docs pass, not a Phase 1 defect.

### Informational

- **I1 — `admin-7` non-GUID principal retained in `test_app_exception_handlers.py:212,230` is correct.** That assertion exercises the exception-handler **logging** path (`v2/src/backend/exception_handlers.py:67-79`), which reads the raw `x-ms-client-principal-id` header directly (documented: "rather than going through the per-router `get_user_id` dependency … exception handlers run outside the request's DI scope"), returns `""` when absent (`test_app_exception_handlers.py:117`), and does **not** apply GUID validation. It is a distinct code path from `get_user_id`, so retaining `admin-7` there is not a violation of the Phase 1 GUID contract or of requirement 4. Phase 1 correctly migrated only the `get_user_id`-adjacent `user-42` case at `:75/:85/:96`.

---

## Coverage assessment

Phase 1 is **fully implemented** and evidenced:

| Plan item | Status | Evidence |
| --- | --- | --- |
| Step 1.1 — rewrite `get_user_id` (Request-only, GUID-or-default, never raises); add `_is_valid_guid` + `_DEFAULT_USER_ID`; remove `_is_valid_principal_id` + `_PRINCIPAL_ID_PATTERN`; `import re`→`import uuid` | Complete | `dependencies.py:19, 273-300`; grep-clean removal |
| Step 1.2 — rewrite the `get_user_id` contract tests across the four files; audit non-GUID principals | Complete | `test_history.py:106-159`; `test_dependencies.py:324-345`; `test_conversation.py:99-101,1026,1099,1113-1127`; `test_app_exception_handlers.py:75-96` |

Changes-log Phase 1 entries (Added/Modified/Removed + the two "Additional or Deviating" notes) all correspond to on-disk state.

## Clarifying questions

None. Phase 1 validation is unambiguous from the available artifacts and source.

## Recommended next validations (not performed this session)

- [ ] Phase 2 — `admin.py` routes → `UserIdDep`; `test_admin.py` override swap + docstring.
- [ ] Phase 3 — role-gate cluster + `require_admin_auth` deletion; `types.py`/`settings.py`/`history.py`/`conversation.py` docstring scrubs; shared-gate exemption removal.
- [ ] Phase 4 — `main.bicep` `AZURE_REQUIRE_ADMIN_AUTH` removal + `AZURE_ENVIRONMENT` retention; `test_main_bicep.py` assertions.
- [ ] Phase 5 — `bugs.md` row correction, worklog, ADR 0031.
- [ ] Docs pass (M1) — refresh `mvp_status.md` / `admin_runtime_config.md` / `development_plan.md` live references to the removed role-gate contract.
