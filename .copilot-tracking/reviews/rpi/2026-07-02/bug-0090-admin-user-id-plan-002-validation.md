<!-- markdownlint-disable-file -->
# RPI Validation: BUG-0090 — Phase 2 "Admin routes → `UserIdDep`"

## Metadata

* **Plan**: .copilot-tracking/plans/2026-07-02/bug-0090-admin-user-id-plan.instructions.md
* **Details**: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Phase 2 = Lines 97-147)
* **Changes log**: .copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md
* **Research**: .copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md
* **Phase**: 2 — Point admin routes at `UserIdDep` (remove the role gate from routes)
* **Validated source**: v2/src/backend/routers/admin.py, v2/tests/backend/test_admin.py
* **Validation date**: 2026-07-02
* **Status**: **PASSED**

## Executive Summary

Phase 2 is **fully and faithfully implemented**. Every `/api/admin/*` route parameter resolves through `UserIdDep`; `AdminUserIdDep`, `requires_role`, `REQUIRE_ADMIN_USER`, `require_admin_auth`, `#39`, and all 401/403 narrative are gone from `admin.py`. The `environment=settings.environment` line in the `AdminStatus` construction is present and untouched. The module docstring is rewritten to the header-GUID contract with zero role-gate / Easy-Auth narrative (Hard Rule #16 clean). `test_admin.py` overrides `get_user_id` (not `REQUIRE_ADMIN_USER`), preserves the status-shape and leak-guard assertions, and the 10 removed 401/403 gate tests are a justified deletion consistent with the plan's uniform-gate-removal intent.

No Critical, Major, or Minor **code defects** found. One **informational** note on tracking-doc line-number drift (not a code defect).

## Requested Checks — Verdict Table

| # | Check | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | Every `/api/admin/*` route param uses `UserIdDep`, not `AdminUserIdDep` (grep `AdminUserIdDep` → 0) | **PASS** | 9/9 routes; `AdminUserIdDep` = 0 in admin.py and 0 across `v2/src/**` |
| 2 | DI import repointed to `UserIdDep`; `requires_role`/`#39` → 0 in admin.py | **PASS** | Import at admin.py:55; `requires_role` = 0, `#39` = 0 |
| 3 | `environment=settings.environment` in `AdminStatus` UNTOUCHED (still present) | **PASS** | admin.py:128 |
| 4 | Module docstring rewritten to header-GUID contract; no role-gate/`#39`/`local-dev`/401/403 narrative | **PASS** | admin.py:1-30 |
| 5 | `test_admin.py` overrides `get_user_id` (not `REQUIRE_ADMIN_USER`); `REQUIRE_ADMIN_USER` → 0; status-shape + leak-guard preserved; 10 deleted 401/403 tests justified | **PASS** | test_admin.py:193; `REQUIRE_ADMIN_USER`/401/403 = 0; status + leak tests intact |

## Check-by-Check Evidence

### Check 1 — Every route uses `UserIdDep`; zero `AdminUserIdDep` — PASS

Nine route decorators, nine `UserIdDep` route parameters — 1:1 coverage.

Route decorators (grep `^@router\.(get|post|patch|delete|put)`):

* [admin.py](../../../../v2/src/backend/routers/admin.py#L104) `GET /status`
* [admin.py](../../../../v2/src/backend/routers/admin.py#L141) `GET /config`
* [admin.py](../../../../v2/src/backend/routers/admin.py#L174) `GET /config/effective`
* [admin.py](../../../../v2/src/backend/routers/admin.py#L258) `PATCH /config`
* [admin.py](../../../../v2/src/backend/routers/admin.py#L400) `GET /documents`
* [admin.py](../../../../v2/src/backend/routers/admin.py#L446) `DELETE /documents/{source}`
* [admin.py](../../../../v2/src/backend/routers/admin.py#L528) `POST` (upload)
* [admin.py](../../../../v2/src/backend/routers/admin.py#L585) `POST` (ingest url)
* [admin.py](../../../../v2/src/backend/routers/admin.py#L646) `POST` (reprocess)

`UserIdDep` route parameters (one per handler): admin.py lines 108, 144, 178, 262, 407, 456, 537, 593, 654.

`AdminUserIdDep` occurrences:

* In admin.py: **0** (grep clean).
* Across `v2/src/**`: **0** (grep clean — confirms the Phase-3 deletion left no dangling reference the Phase-2 swap depends on).

Satisfies details Step 2.1 success criterion "No `AdminUserIdDep` reference remains in `admin.py`" and "All `/api/admin/*` routes depend on `UserIdDep`". Matches research §"Selected approach" (uniform `UserIdDep`) and rejects Alt C (partial relax) — all routes swapped, not just `/status`.

### Check 2 — Import repointed; no `requires_role`/`#39` — PASS

DI import block repoints to `UserIdDep`:

* [admin.py](../../../../v2/src/backend/routers/admin.py#L46-L56) — `from backend.dependencies import (... SettingsDep, UserIdDep)`; `UserIdDep` at line 55.

Banned-symbol grep in admin.py for `AdminUserIdDep|requires_role|REQUIRE_ADMIN_USER|#39|require_admin_auth|local-dev|401|403` returned **0 matches** — every one of these is absent from the module.

### Check 3 — `environment=settings.environment` untouched — PASS

* [admin.py](../../../../v2/src/backend/routers/admin.py#L128) — `environment=settings.environment,` inside the `AdminStatus(...)` construction in `status_endpoint`.

The field is present and semantically unchanged, feeding `AdminStatus.environment` exactly as PD-02 requires (details "Selected decision (PD-02): keep the `environment` field"). Satisfies the Step 2.1 criterion "`AdminStatus.environment` still populated from `settings.environment`".

> Informational (not a defect): the details doc cites this as "line 138" (details Line ~120 target-code list and Step 2.1 checklist). The actual line is **128**. The 10-line upward drift is a mechanical consequence of the module docstring being shortened during the Phase-2 rewrite (the old 29-46 role-gate narrative was removed). The code itself is correct and untouched; only the tracking-doc line number is stale. Recorded for evidence discipline; no action required for Phase 2.

### Check 4 — Module docstring rewritten; Hard Rule #16 clean — PASS

* [admin.py](../../../../v2/src/backend/routers/admin.py#L1-L30) — module docstring.

The docstring now states every route resolves through `backend.dependencies.UserIdDep`, "which reads the `x-ms-client-principal-id` header and returns the caller's GUID when it is present and well-formed, or the anonymous default GUID otherwise. There is no role gate and no Easy Auth requirement — the router never rejects a request on identity grounds."

No `requires_role`, no `#39`, no `local-dev`, no 401/403 narrative in the module docstring — present-tense, contract-only. Satisfies Step 2.1 "Module docstring reflects the header-GUID contract with no role/Easy-Auth narrative" and Hard Rule #16.

Per the validation directive: per-route docstrings still carry pre-existing `#35x` tokens (e.g. `#35b` at [admin.py](../../../../v2/src/backend/routers/admin.py#L148), `#35c`/`#35e`/`#35f` on later handlers). These are logged follow-on **WI-04**, **NOT a Phase-2 defect** — the module docstring is the Phase-2 surface and it is clean. Not flagged.

### Check 5 — Test overrides `get_user_id`; assertions preserved; deletions justified — PASS

**Override repointed** — the fixture overrides `get_user_id`, not `REQUIRE_ADMIN_USER`:

* [test_admin.py](../../../../v2/tests/backend/test_admin.py#L193) — `app.dependency_overrides[get_user_id] = lambda: _FIXED_USER_ID`
* [test_admin.py](../../../../v2/tests/backend/test_admin.py#L47) — `get_user_id` imported from `backend.dependencies`
* [test_admin.py](../../../../v2/tests/backend/test_admin.py#L63-L65) — `_FIXED_USER_ID = "3f2504e0-4f89-41d3-9a0c-0305e82c3301"` (valid GUID)

`REQUIRE_ADMIN_USER` / `AdminUserIdDep` / `requires_role` / `require_admin_auth` / `401` / `403` occurrences in test_admin.py: **0**. All 8 grep hits for the combined pattern resolve to `get_user_id` (lines 7, 9, 47, 63, 191, 193, 325, 779) — the new contract only.

**Module docstring refreshed** — [test_admin.py](../../../../v2/tests/backend/test_admin.py#L1-L11) describes the `get_user_id` / `UserIdDep` GUID contract and "there is no role gate"; no stale `#39` / `REQUIRE_ADMIN_USER` / `test_requires_role_*` references. Satisfies Step 2.2.

**Status-shape assertions preserved** — the full `test_status_*` suite is intact (test_admin.py lines 242, 252, 269, 283, 300, 310, 337, 359, 372):

* [test_admin.py](../../../../v2/tests/backend/test_admin.py#L310-L334) `test_status_maps_orchestrator_db_index_environment` — sets `environment="production"`, and its comment/assertions prove the request now **succeeds** (200) and `body["environment"] == "production"`. This is the direct behavioral proof that the BUG-0090 401 no longer exists: the exact scenario that used to raise (production + no claims blob) now returns the status snapshot.

**Leak-guard preserved** — [test_admin.py](../../../../v2/tests/backend/test_admin.py#L390-L421) `test_status_does_not_leak_sensitive_settings` is intact, parametrized over four `DO-NOT-LEAK` markers (tenant / uami / cosmos / api-version), asserting `marker not in resp.text`.

**Deleted 401/403 tests are justified** — the changes log ("Removed", Phase 2) lists the 10 route-level role-gate tests (`test_*_requires_easy_auth_in_production`, `test_status_endpoint_returns_403/200_*`) deleted because they asserted 401/403 that the gate removal makes unreachable. This is consistent with the plan's uniform-gate-removal intent (Objectives → "Unify chat/history/admin under one `UserIdDep`"; research §"Selected approach" — "the 401 is impossible"). The observable outcome corroborates the deletion: zero remaining 401/403 assertions, and `test_status_maps_orchestrator_db_index_environment` now asserts 200 in production. Deleting tests for behavior that no longer exists matches the reduce-code-debt directive, not a coverage regression.

## Cross-Artifact Coverage Assessment

| Plan/Details Phase-2 item | Implemented | Evidence |
|---|---|---|
| Step 2.1: `AdminUserIdDep` → `UserIdDep` on all routes | Yes | 9/9 route params at UserIdDep; 0 `AdminUserIdDep` |
| Step 2.1: import repointed | Yes | admin.py:55 |
| Step 2.1: full module docstring rewrite (drop `requires_role`/`#39`/`local-dev`) | Yes | admin.py:1-30 |
| Step 2.1: keep `environment=settings.environment` | Yes | admin.py:128 |
| Step 2.2: override `get_user_id` not `REQUIRE_ADMIN_USER` | Yes | test_admin.py:193; drop confirmed (0 hits) |
| Step 2.2: refresh test module docstring | Yes | test_admin.py:1-11 |
| Step 2.2: keep status-shape + leak-guard assertions | Yes | test_status_* suite + leak-guard intact |
| Changes log: 9 routes swapped, import repointed, docstring rewritten, `environment` untouched | Matches source | grep + line evidence above |
| Changes log: 10 dead 401/403 route tests removed (justified) | Consistent | 0 residual 401/403; production test asserts 200 |

Phase-2 coverage: **complete**. No plan item is missing a corresponding change; no change contradicts the plan or research.

## Findings by Severity

### Critical

* None.

### Major

* None.

### Minor

* None (no code defect).

### Informational

* **INFO-01 — tracking-doc line-number drift (not a code defect).** Details Phase-2 cites `environment=settings.environment` at "line 138"; the actual location is admin.py:128 (10-line upward drift from the docstring shortening during the same phase). Source is correct and untouched; only the plan/details line reference is stale. Optional: refresh the details line reference in a future doc-maintenance pass. No Phase-2 remediation required.

## Recommended Next Validations (not run this session)

* [ ] Phase 1 — Minimal `get_user_id` (dependencies.py + the four `get_user_id` contract test files).
* [ ] Phase 3 — Deletion of the role-gate cluster (`requires_role`, `_decode_easy_auth_principal`, `_extract_roles`, `REQUIRE_ADMIN_USER`, `AdminUserIdDep`, `_LOCAL_DEV_USER`, `_PRINCIPAL_HEADER`) from dependencies.py + `require_admin_auth` from settings.py + types.py/history.py/conversation.py doc scrubs; tree-wide grep-clean.
* [ ] Phase 4 — Bicep `AZURE_REQUIRE_ADMIN_AUTH` removal + `AZURE_ENVIRONMENT` retention + infra test.
* [ ] Phase 5 — bugs.md registry row, worklog, ADR 0031.
* [ ] Phase 6 — full-suite run, gates, and the pending live `/api/admin/status` 200 verification (Step 6.3, gated on user go-ahead per Hard Rule #10).
* [ ] Follow-on WI-04 — per-route `#35x` docstring token scrub in admin.py (tracked, out of Phase-2 scope).

## Clarifying Questions

* None. All Phase-2 checks resolved from source + tracking artifacts without ambiguity.
