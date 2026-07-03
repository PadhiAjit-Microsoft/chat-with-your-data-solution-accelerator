<!-- markdownlint-disable-file -->
# Release Changes: BUG-0090 — Admin 401 & user_id header handling

**Related Plan**: bug-0090-admin-user-id-plan.instructions.md
**Implementation Date**: 2026-07-02

## Summary

Collapse the CWYD v2 backend to one minimal `get_user_id` (present + valid GUID → use, else default GUID; never raises), swap every `/api/admin/*` route to it, and delete the entire Easy Auth admin role gate + `require_admin_auth` setting + the `AZURE_REQUIRE_ADMIN_AUTH` Bicep env var — so `/api/admin/status` can no longer return the BUG-0090 401. Mirrors MACAE; frontend unchanged; `environment` field retained.

## Changes

### Added

* v2/docs/adr/0031-backend-admin-auth-header-only-ingress-enforced.md - new ADR documenting the header-only/ingress-enforced admin auth posture, security tradeoff, and revert path. [Phase 5]

### Modified

* v2/src/backend/dependencies.py - `get_user_id` rewritten to a Request-only GUID contract (present + valid GUID → verbatim, else default GUID `00000000-0000-0000-0000-000000000000`, never raises); `import re` → `import uuid`; added `_DEFAULT_USER_ID` + `_is_valid_guid`. [Phase 1]
* v2/tests/backend/test_history.py - `test_get_user_id_*` suite rewritten to the GUID contract; `_TEST_USER_ID` GUID constant threaded through the `get_user_id` override + 8 router assertions. [Phase 1]
* v2/tests/backend/test_dependencies.py - the second `get_user_id` block rewritten to the GUID contract (no `settings` arg); `requires_role` suite untouched (deleted in Phase 3). [Phase 1]
* v2/tests/backend/test_conversation.py - `local-dev` fallback assertions → default GUID; principal-echo test uses a valid GUID header. [Phase 1]
* v2/tests/backend/test_app_exception_handlers.py - `user-42` → valid GUID (header + echo assertion). [Phase 1]
* v2/infra/main.bicep - removed backend `AZURE_REQUIRE_ADMIN_AUTH` env entry + comment; rewrote backend `AZURE_ENVIRONMENT` comment (kept the entry); functions block untouched. [Phase 4]
* v2/tests/infra/test_main_bicep.py - added `test_backend_aca_env_drops_require_admin_auth_keeps_environment`. [Phase 4]
* v2/src/backend/routers/admin.py - all 9 `/api/admin/*` routes swapped `AdminUserIdDep` → `UserIdDep`; import repointed; full module docstring rewritten to the header-GUID contract (role-gate/`#39`/`local-dev`/401/403 narrative removed); `environment=settings.environment` untouched. [Phase 2]
* v2/tests/backend/test_admin.py - fixture override → `get_user_id` returning a fixed GUID; dropped `REQUIRE_ADMIN_USER` + orphaned `base64`/`json` imports; module docstring refreshed. [Phase 2]
* v2/src/backend/core/settings.py - deleted `require_admin_auth` field + comment; scrubbed the `environment` field comment + `Environment` enum docstring of auth narrative (kept the field + enum). [Phase 3]
* v2/src/backend/core/types.py - rewrote `RuntimeConfig.updated_by` (line 320) AND `AdminAuditEntry.actor` (line 352) docstrings to reference `get_user_id`/`UserIdDep` instead of the deleted `REQUIRE_ADMIN_USER`/`requires_role`. [Phase 3]
* v2/src/backend/routers/history.py - module docstring scrubbed to the default-GUID contract (doc-only). [Phase 3]
* v2/src/backend/routers/conversation.py - persistence comment scrubbed of `local-dev` (doc-only). [Phase 3]
* v2/tests/backend/core/test_settings.py - deleted `test_require_admin_auth_defaults_to_false` + `test_require_admin_auth_env_override_enables_wall`. [Phase 3]
* v2/tests/shared/test_no_anonymous_dict_returns.py - removed the `_decode_easy_auth_principal` Hard-Rule-#15 allow-list entry + docstring bullet. [Phase 3]
* v2/docs/bugs.md - rewrote the BUG-0090 registry row (corrected root cause = the `requires_role("admin")` gate; delivered fix = single GUID `get_user_id` + gate/`require_admin_auth`/bicep-var deletion); status kept `open` pending Phase 6. [Phase 5]
* v2/docs/worklog/2026-07-02.md - appended the BUG-0090 fix + corrected root cause + security tradeoff + frontend no-op verification. [Phase 5]
* v2/docs/adr/README.md - added the ADR 0031 index row. [Phase 5]
* v2/src/frontend/src/api/auth.tsx - doc-only: corrected the now-stale `PRINCIPAL_ID_HEADER` docstring (removed the "admin RBAC anchored on backend Easy Auth claims" line the backend change made false; describes GUID-validate + ingress-level auth). [Phase 6 / WI-05]
* v2/docs/mvp_status.md - refreshed the executive-snapshot line, §1 tenant-isolation, §3 admin flow + mermaid, the entire §4 auth-flow section (+ flowchart + gaps), and completion-plan task A2 to the header-GUID / ingress-enforced posture (removed `requires_role`/`AdminUserIdDep`/RBAC/claims-blob/401-403/local-dev narrative). [Review F-1]
* v2/docs/admin_runtime_config.md - refreshed §1.7, §2 intro, five per-route status-code lines (dropped the removed 401-auth/403-role codes), and the `updated_by` provenance to the header-GUID contract. [Review F-1]
* v2/docs/project_status.md - updated two live-status rows: the `requires_role` RBAC-gate row → "removed (BUG-0090)"; the fail-closed `get_user_id` row → the header-GUID/never-401 contract. [Review F-1]
* v2/tests/shared/test_no_anonymous_dict_returns.py - fixed the merged allow-list docstring bullets (line break lost when the `_decode_easy_auth_principal` exemption was removed in Phase 3). [Review F-2]

### Removed

* v2/src/backend/dependencies.py: `_is_valid_principal_id`, `_PRINCIPAL_ID_PATTERN` (allowlist superseded by `_is_valid_guid`); unused `import re`. [Phase 1]
* v2/tests/backend/test_history.py: `test_is_valid_principal_id_accepts_well_formed`, `test_is_valid_principal_id_rejects_malformed`, `test_get_user_id_rejects_malformed_principal_id` (subject symbol removed). [Phase 1]
* v2/tests/backend/test_dependencies.py: obsolete `test_get_user_id_raises_401_*` / `test_get_user_id_falls_back_to_local_dev_when_open_auth_in_prod`. [Phase 1]
* v2/tests/backend/test_admin.py: 10 now-dead route-level role-gate tests (`test_*_requires_easy_auth_in_production`, `test_status_endpoint_returns_403/200_*`) that asserted 401/403 no longer produced after the gate removal. [Phase 2]
* v2/src/backend/dependencies.py: the entire Easy Auth role-gate cluster — `requires_role` (+ inner `_checker`), `_decode_easy_auth_principal`, `_extract_roles`, `REQUIRE_ADMIN_USER`, `AdminUserIdDep`, `_PRINCIPAL_HEADER`, `_LOCAL_DEV_USER`, `_ROLE_TYP_SHORT`/`_ROLE_TYP_FULL`; dead imports (`base64`, `binascii`, `json`, `Callable`, `Any`, `cast`, `HTTPException`, `status`, `Environment`); trimmed `__all__`. [Phase 3]
* v2/src/backend/core/settings.py: the `require_admin_auth: bool = False` field + comment. [Phase 3]
* v2/tests/backend/test_dependencies.py: the `test_requires_role_*` suite + claims helpers + dead imports. [Phase 3]

## Additional or Deviating Changes

* Phase 1: introduced a `_TEST_USER_ID` GUID constant in test_history.py (details called the line-94 override change "optional but preferred"; required to keep 8 dependent router assertions green after switching the override to a GUID).
  * Reason: 8 downstream router assertions depend on the override value; a shared GUID constant keeps them consistent.
* Phase 1: deleted 3 allowlist-bound tests rather than migrating them (their subject `_is_valid_principal_id` was removed; the malformed→401 behavior no longer exists).
  * Reason: the tested behavior was deleted; the clean 4-test GUID suite supersedes them.
* Phase 4: `v2/infra/main.json` (untracked, gitignored compiled ARM artifact) still carries the `AZURE_REQUIRE_ADMIN_AUTH` literal at compiled line 48372; regeneration deferred to Phase 6 / next `azd` build (untracked → cannot leak).
* Observed (pre-existing, OUT OF SCOPE): the env-ID gate fails on `.copilot-tracking/research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md` (a BUG-0055 artifact leaking `AZURE_ENV_NAME`); NOT introduced by this work; neither BUG-0090 edit appears in the hit list.
  * RESOLVED in Phase 6: scrubbed the leaked env-name literal on line 6 of that BUG-0055 research doc to the `<AZD_ENV_NAME>` placeholder (binding env-ID rule; unblocks the green suite). Out-of-scope opportunistic fix.
* Review-finding remediation (2026-07-03): addressed the Task Reviewer's findings.
  * F-1 scope was broader than the spot-checked lines — refreshed ALL current-state / live-status admin-auth references across mvp_status.md, admin_runtime_config.md, and project_status.md, not just the three cited lines.
  * Deliberately LEFT the historical records that correctly describe past work (rewriting them would be revisionism): `development_plan.md` §0.1 ledger rows (cleared `#39` + `U-P7-ROUTER-CLEAN`), `bugs.md` BUG-0087/0088/0089 history, ADRs 0023/0024 (superseded records), and `frontend-user-identity-plan.md` (executed plan). Their references to the removed symbols are archaeological, not current-state claims.
  * Frontend admin-button-visibility claims (mvp_status.md lines 30/127, completion-plan A3) left as-is — client-side `/.auth/me` UX independent of the removed backend gate.
  * F-3 (plan/details line-citation drift): no action, per the review's own guidance ("no action required unless the tracking docs are reused").

## Release Summary

**Local implementation complete and green across all 6 phases; live deploy (Step 6.3) pending user go-ahead per Hard Rule #10.**

Backend source (5 files + 1 frontend): `dependencies.py` (`get_user_id` collapsed to the minimal GUID contract; entire Easy Auth role-gate cluster deleted), `routers/admin.py` (all 9 admin routes → `UserIdDep`, docstring rewritten), `core/settings.py` (`require_admin_auth` deleted; `environment` kept + comment scrubbed), `core/types.py` (2 docstrings), `routers/history.py` + `routers/conversation.py` (doc scrubs), `frontend/src/api/auth.tsx` (1 stale docstring, doc-only).

Backend tests (6): `test_dependencies.py`, `test_history.py`, `test_conversation.py`, `test_app_exception_handlers.py`, `test_admin.py`, `core/test_settings.py`. Shared gate (1): `test_no_anonymous_dict_returns.py`.

Infra (2): `main.bicep` (backend `AZURE_REQUIRE_ADMIN_AUTH` removed; `AZURE_ENVIRONMENT` kept + re-commented), `tests/infra/test_main_bicep.py`.

Docs (4): `bugs.md` (BUG-0090 corrected, status `open`), `worklog/2026-07-02.md`, `adr/0031-backend-admin-auth-header-only-ingress-enforced.md` (new), `adr/README.md`. Out-of-scope cleanup (1): scrubbed a pre-existing BUG-0055 env-ID leak.

**Validation:** backend 2176 passed / 1 skipped; infra 39 passed; shared 1039 passed (env-ID gate green); `az bicep build` exit 0; `pyright`/`get_errors` clean on all touched files; grep-clean for every deleted symbol across `v2/src` + `v2/tests`.

**Deployment notes:** The live deploy (`azd provision` / `azd deploy backend` from a `v2`-cwd terminal) + `GET /api/admin/status` 200 verification is the only remaining step and needs explicit go-ahead. `v2/infra/main.json` (untracked ARM artifact) regenerates on deploy. BUG-0090 stays `open` until live-verified, then flips to `fixed`.
