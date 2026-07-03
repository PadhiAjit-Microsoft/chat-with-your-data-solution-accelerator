---
applyTo: '.copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md'
---
<!-- markdownlint-disable-file -->
# Implementation Plan: BUG-0090 — Admin 401 & user_id header handling

## Overview

Collapse the CWYD v2 backend to one minimal `get_user_id` (present + valid GUID → use, else default GUID; never raises), swap every `/api/admin/*` route to it, delete the entire Easy Auth admin role gate + `require_admin_auth` setting + the `AZURE_REQUIRE_ADMIN_AUTH` Bicep env var, so `/api/admin/status` can no longer 401 — mirroring MACAE.

## Objectives

### User Requirements

* Backend must ONLY check that the `user_id` header is present and is a valid GUID — nothing more. — Source: user request 2026-07-02.
* The `/api/admin/status` 401 "shouldn't even exist" — remove that gate. — Source: user request 2026-07-02.
* Frontend auth ON → real user_id + initials; auth OFF → default GUID + `G` initial; always pass user_id in headers. — Source: user request 2026-07-02.
* Mirror how MACAE handles `user_id` at the backend; any extra safety is unnecessary. — Source: user request 2026-07-02.

### Derived Objectives

* Unify chat/history/admin under one `UserIdDep` (uniform contract). — Derived from: user's rule is general ("the backend should only check…"), so it must apply to every route, not just `/status` (rejects Alt C).
* Delete the now-dead role-gate cluster + `require_admin_auth` + Bicep env var. — Derived from: cleanup-before-next-step / reduce-code-debt memory; leaving tested-but-dead code rots the tree.
* Keep the `environment` field + `Environment` enum. — Derived from: `admin.py:138` still feeds `AdminStatus.environment` (bicep-env research Gap 2) — the minimal-change path.
* No frontend change. — Derived from: frontend research proves it already sends the header, defaults to the all-zeros GUID, and renders `G`.

## Context Summary

### Project Files

* v2/src/backend/dependencies.py - rewrite `get_user_id`; delete the role-gate cluster (`requires_role`, `_checker`, `_decode_easy_auth_principal`, `_extract_roles`, `REQUIRE_ADMIN_USER`, `AdminUserIdDep`, `_PRINCIPAL_HEADER`, role-typ constants, `_LOCAL_DEV_USER`, `_is_valid_principal_id`, `_PRINCIPAL_ID_PATTERN`).
* v2/src/backend/routers/admin.py - `AdminUserIdDep` → `UserIdDep` on all routes; keep `environment=settings.environment` (line 138); rewrite the FULL docstring (drop `requires_role`/`#39`/`local-dev` narrative).
* v2/src/backend/core/settings.py - delete `require_admin_auth` (line 543); keep `environment` (533) + `Environment` enum (41-52).
* v2/src/backend/core/types.py - rewrite the `RuntimeConfig.updated_by` docstring (line 320) that references the deleted `REQUIRE_ADMIN_USER` / `requires_role`.
* v2/src/backend/routers/history.py, v2/src/backend/routers/conversation.py - doc-only: scrub stale `local-dev`/401 `get_user_id` narrative from docstrings (Hard Rule #16).
* v2/infra/main.bicep - delete backend `AZURE_REQUIRE_ADMIN_AUTH` (1813); keep + re-comment backend `AZURE_ENVIRONMENT` (1805); functions `AZURE_ENVIRONMENT` (2160) untouched.
* v2/tests/backend/test_dependencies.py - rewrite the `get_user_id` block (Phase 1); delete the `test_requires_role_*` suite + imports (Phase 3).
* v2/tests/backend/test_history.py - rewrite the authoritative `test_get_user_id_*` suite (Phase 1).
* v2/tests/backend/test_conversation.py, v2/tests/backend/test_app_exception_handlers.py - default-GUID fallback + GUID-principal echo (Phase 1).
* v2/tests/backend/test_admin.py - override `get_user_id` not `REQUIRE_ADMIN_USER`; drop `require_admin_auth`; refresh docstring.
* v2/tests/backend/core/test_settings.py - delete the two `require_admin_auth` tests.
* v2/tests/shared/test_no_anonymous_dict_returns.py - remove the `_decode_easy_auth_principal` Hard-Rule-#15 exemption.
* v2/tests/infra/test_main_bicep.py - assert `AZURE_REQUIRE_ADMIN_AUTH` absence + `AZURE_ENVIRONMENT` presence.
* v2/docs/bugs.md, v2/docs/worklog/2026-07-02.md, v2/docs/adr/ - tracking + posture ADR.

### References

* .copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md - selected approach, alternatives, security tradeoff.
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-backend-auth-wiring-research.md - backend wiring + file:line index + test inventory.
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-macae-user-id-pattern-research.md - MACAE contract.
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-frontend-user-id-research.md - frontend already compliant.
* .copilot-tracking/research/subagents/2026-07-02/bug-0090-bicep-env-and-environment-usage-research.md - Bicep env wiring + `environment` consumer audit.

### Standards References

* .github/copilot-instructions.md - Hard Rule #1 (one unit/turn), #2 (test-first), #10 (structural-change confirmation), #16 (no process narrative in `v2/src/**`), #17 (imports-at-top), #18 (no env IDs), #19 (durable bugs.md + worklog).
* .github/instructions/v2-backend.instructions.md, .github/instructions/v2-backend-core.instructions.md - router/dependency/settings conventions.

## Implementation Checklist

### [ ] Implementation Phase 1: Minimal `get_user_id`

<!-- parallelizable: false -->

* [ ] Step 1.1: Rewrite `get_user_id` (Request-only; valid GUID → use, else default GUID, never raise); add `_is_valid_guid` + `_DEFAULT_USER_ID`; remove `_is_valid_principal_id` + `_PRINCIPAL_ID_PATTERN`
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 20-67)
* [ ] Step 1.2: Rewrite EVERY `get_user_id` contract test (test_history.py, test_dependencies.py, test_conversation.py, test_app_exception_handlers.py); audit non-GUID principals
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 68-96)

### [ ] Implementation Phase 2: Admin routes → `UserIdDep`

<!-- parallelizable: false -->

* [ ] Step 2.1: Swap `AdminUserIdDep` → `UserIdDep` on all `/api/admin/*` routes; update import + rewrite full docstring; keep line 138
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 101-129)
* [ ] Step 2.2: Update `test_admin.py` to override `get_user_id` instead of `REQUIRE_ADMIN_USER`; refresh its docstring
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 130-147)

### [ ] Implementation Phase 3: Delete the dead role-gate cluster + `require_admin_auth`

<!-- parallelizable: false -->

* [ ] Step 3.1: Grep-verify zero prod callers; delete the role-gate cluster from `dependencies.py` + `require_admin_auth` from `settings.py` + rewrite the `types.py` docstring + scrub stale narrative (settings/history/conversation); keep `environment`/`Environment`
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 152-194)
* [ ] Step 3.2: Delete the role-gate + `require_admin_auth` tests + remove the `_decode_easy_auth_principal` exemption in the shared gate
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 195-215)

### [ ] Implementation Phase 4: Bicep — remove `AZURE_REQUIRE_ADMIN_AUTH`, refresh `AZURE_ENVIRONMENT` comment

<!-- parallelizable: true -->

* [ ] Step 4.1: Delete the backend `AZURE_REQUIRE_ADMIN_AUTH` env entry + comment; rewrite the backend `AZURE_ENVIRONMENT` comment (keep the entry)
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 220-246)
* [ ] Step 4.2: Update/add infra tests asserting `AZURE_REQUIRE_ADMIN_AUTH` absence + `AZURE_ENVIRONMENT` presence
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 247-263)

### [ ] Implementation Phase 5: Documentation

<!-- parallelizable: false -->

* [ ] Step 5.1: Correct the BUG-0090 registry row + append a worklog entry (flip status at Phase 6)
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 268-286)
* [ ] Step 5.2: Author (or amend) an ADR for the auth-posture decision + revert path
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 287-303)
* [ ] Step 5.3: Verify the frontend is already compliant (no code change)
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 304-319)

### [ ] Implementation Phase 6: Validation

<!-- parallelizable: false -->

* [ ] Step 6.1: Run backend + infra tests, bicep build, env-ID gate + shared invariant gates
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 324-332)
* [ ] Step 6.2: Fix minor validation issues (pyright/ruff/test) scoped to touched files
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 333-336)
* [ ] Step 6.3: Live deploy + verify `/api/admin/status` returns 200 (gated on user go-ahead per Hard Rule #10); flip BUG-0090 → fixed
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 337-340)
* [ ] Step 6.4: Report blocking issues with next steps
  * Details: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Lines 341-354)

## Planning Log

See .copilot-tracking/plans/logs/2026-07-02/bug-0090-admin-user-id-log.md for discrepancy tracking, implementation paths considered, and suggested follow-on work.

## Dependencies

* Python 3.11+ venv at `v2/.venv` (`v2\.venv\Scripts\python.exe`); pytest.
* Azure CLI / `az bicep` for the bicep build; `azd` (run from v2 cwd) for the optional live deploy.
* FastAPI + Pydantic v2 (existing stack).

## Success Criteria

* `/api/admin/status` and every `/api/admin/*` route can no longer return the Easy-Auth 401. — Traces to: user request; research "Corrected root cause".
* `get_user_id` is a single dependency: header present + valid GUID → use; else default GUID; never raises. — Traces to: user request ("only check present + valid GUID"); MACAE pattern.
* Role-gate cluster + `require_admin_auth` + `AZURE_REQUIRE_ADMIN_AUTH` fully removed (no dead code). — Traces to: reduce-code-debt memory.
* `environment` field retained (feeds `AdminStatus.environment`); frontend unchanged. — Traces to: bicep-env research Gap 2; frontend research.
* BUG-0090 registry + worklog + ADR recorded; backend + infra tests, bicep build, env-ID + shared gates all green. — Traces to: Hard Rule #19.
