<!-- markdownlint-disable-file -->
# Task Review: BUG-0090 — Admin 401 & user_id header handling

## Review Metadata

* **Review date**: 2026-07-02
* **Related plan**: .copilot-tracking/plans/2026-07-02/bug-0090-admin-user-id-plan.instructions.md
* **Changes log**: .copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md
* **Research document**: .copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md
* **Planning log**: .copilot-tracking/plans/logs/2026-07-02/bug-0090-admin-user-id-log.md
* **Reviewer scope**: All 6 implementation phases (Phase 6.3 live deploy is intentionally deferred, gated on user go-ahead per Hard Rule #10).

## Summary

The plan is faithfully implemented across all delivered scope. Every RPI phase validator PASSED against the actual source (not just the changes log). The full backend + infra + shared suite is green (2177 passed, 1 skipped), all shared invariant gates pass, and every deleted symbol is grep-clean across `v2/src` + `v2/tests`. No Critical or Major findings. The only substantive follow-up is stale references to the removed auth contract in three live product docs (not in the plan's Phase-5 doc scope). The live deploy remains correctly gated.

| Severity | Count |
|----------|-------|
| Critical | 0 |
| Major    | 0 |
| Minor    | 3 |
| Informational / Accepted | 3 |

## RPI Validation Findings (per phase)

| Phase | Status | Evidence | Findings |
|-------|--------|----------|----------|
| 1 — Minimal `get_user_id` | ✅ PASS | `dependencies.py:286` Request-only GUID contract; `_is_valid_guid`/`_DEFAULT_USER_ID` present; `_is_valid_principal_id`/`_PRINCIPAL_ID_PATTERN` grep-clean; 4 test files migrated | Minor F-1 (stale live docs, cross-phase); Info: exception-handler raw-header logger correctly bypasses `get_user_id` |
| 2 — Admin routes → `UserIdDep` | ✅ PASS | 9/9 routes on `UserIdDep`; `AdminUserIdDep`=0 in `v2/src/**`; `environment=settings.environment` intact (admin.py:128); module docstring scrubbed | Minor F-3 (line-citation drift :138→:128) |
| 3 — Delete role-gate cluster | ✅ PASS | Tree-wide grep-clean for all deleted symbols incl. docstrings; `environment`/`Environment` retained; dead imports removed; `get_errors` clean on 6 files | Minor F-2 (cosmetic docstring bullet); Minor F-3 (line drift) |
| 4 — Bicep | ✅ PASS | `AZURE_REQUIRE_ADMIN_AUTH`=0 in main.bicep; backend `AZURE_ENVIRONMENT` retained + re-commented; functions block byte-identical; regression test added (39 passed) | Info MN-01 (untracked main.json still has literal); Minor F-3 (line drift) |
| 5 — Documentation | ✅ PASS | BUG-0090 root cause corrected + status still `open`; adjacent rows intact; worklog appended; ADR 0031 + README row present; no env IDs | Minor F-1 (stale live docs, cross-phase) |
| 6 — Validation | ✅ PASS (delivered) · ⏸ 6.3 deferred | auth.tsx docstring corrected (doc-only, `get_errors` clean); BUG-0055 leak scrubbed to `<AZD_ENV_NAME>`; BUG-0090 correctly left `open`; no deploy executed; WI-04/05/06/07 recorded | Info (suite re-run is a QV step — done, see below) |

## Implementation Quality Findings

_The Implementation Validator subagent lacks file access in this environment; quality was assessed directly from the source-verified RPI results, the green test suite, and the passing invariant gates._

* **Architecture / design — PASS.** Faithful to the selected approach: one minimal `get_user_id`, the entire Easy Auth role gate removed, all admin routes unified on `UserIdDep`. Mirrors MACAE. `environment` field correctly retained (feeds `AdminStatus.environment`). No registry / plug-and-play surface disturbed.
* **Code standards — PASS.** The shared invariant gates all pass (part of the 2177): imports-at-top (#17), no-process-narrative-in-src (#16), no-anonymous-dict-returns (#15), init-marker (#13). `pyright`/`get_errors` clean on every touched file (RPI-confirmed). `Environment` kept as `StrEnum` (#11).
* **Test coverage — PASS.** The `get_user_id` GUID contract has explicit tests (valid GUID → verbatim; missing/blank/non-GUID → default GUID); the admin 200-path is proven by `test_status_maps_orchestrator_db_index_environment` (environment=production now asserts 200, directly disproving the BUG-0090 401); deleted symbols have deleted tests (no orphans); infra regression test added.
* **Dead-code hygiene — PASS.** Role-gate cluster + dead imports + orphaned tests fully removed (reduce-code-debt satisfied); the Hard-Rule-#15 allow-list exemption for the deleted `_decode_easy_auth_principal` was removed.
* **Security — ACCEPTED RISK (documented).** `x-ms-client-principal-id` is client-forgeable; with the gate removed, admin routes (incl. writes) are open to anyone who can reach the backend FQDN. This is a deliberate, user-directed decision matching MACAE, documented in ADR 0031 with the ingress-level mitigation path (WI-01). Not a defect given the directive; flagged as an accepted risk the operator must close at the ingress layer before exposing the backend publicly.

## Validation Command Outputs

| Command | Result |
|---------|--------|
| `pytest v2/tests/backend v2/tests/infra v2/tests/shared -q` | **2177 passed, 1 skipped, 31 warnings** (warnings are pre-existing agent_framework experimental + FastAPI 422 deprecation) |
| shared invariant gates (subset of above) | all green (imports-at-top, no-process-narrative, no-anonymous-dict, init-marker) |
| env-ID gate (`test_no_env_specific_content`) | **green** (BUG-0055 leak scrubbed) |
| `az bicep build v2/infra/main.bicep --stdout` | **exit 0** |
| `get_errors` (touched files, via RPI) | clean |

## Missing Work & Deviations

* **Deferred by design (not missing):** Step 6.3 live deploy (`azd provision`/`azd deploy backend`) + `/api/admin/status` 200 verification + flip BUG-0090 → `fixed`. Gated on user go-ahead (Hard Rule #10).
* **Deviations (all justified, logged in the planning log):**
  * DD-08 — `_TEST_USER_ID` GUID constant + deletion of 3 allowlist-bound tests (subject symbol removed).
  * DD-09 — `v2/infra/main.json` (untracked) not regenerated; refreshes on next `azd` build.
  * DD-10 — a second stale `REQUIRE_ADMIN_USER` reference (`types.py` `AdminAuditEntry.actor`) fixed beyond the plan's single-line pointer.
  * Out-of-scope: scrubbed a pre-existing BUG-0055 env-ID leak; corrected a now-stale `auth.tsx` docstring (WI-05).

## Follow-Up Work

### Discovered during review (recommended)

* **F-1 (Minor) — stale live docs.** `v2/docs/mvp_status.md` (L125,148), `v2/docs/admin_runtime_config.md` (L20), `v2/docs/development_plan.md` (L115,169) still describe the removed `requires_role`/`AdminUserIdDep`/`require_admin_auth` contract. Refresh to the header-GUID / ingress-enforced posture. (Not in the plan's Phase-5 doc scope.)
  * ✅ **RESOLVED (2026-07-03).** Refreshed ALL current-state admin-auth references across `mvp_status.md` (executive snapshot, §1 tenant-isolation, §3 admin flow + mermaid, the whole §4 auth-flow section + flowchart + gaps, completion-plan A2) and `admin_runtime_config.md` (§1.7, §2 intro, 5 per-route status-code lines, `updated_by`). `development_plan.md` L115/169 LEFT unchanged — historical §0.1 debt-ledger rows (cleared `#39` + `U-P7-ROUTER-CLEAN`), not current-state claims. Frontend admin-button-visibility claims left as client-side UX.
* **F-2 (Minor) — cosmetic.** `v2/tests/shared/test_no_anonymous_dict_returns.py:40` docstring bullet list lost a line break where the deleted `_decode_easy_auth_principal` bullet sat. Enforced frozenset is unaffected.
  * ✅ **RESOLVED (2026-07-03).** Restored the line break between the two merged allow-list bullets; gate re-run green (2 passed, 1 skipped).
* **F-3 (Minor) — tracking line-citation drift.** Plan/details cite `admin.py:138` (actual :128), bicep `:1805`/`:2160` (actual :1797/:2144). Source is correct; only the tracking docs' line numbers drifted (mechanical, from docstring/entry shrink). No action required unless the tracking docs are reused.
  * ☑ **NO ACTION (2026-07-03)** — per this finding's own guidance; implementation is complete so the plan/details will not be reused.

### Deferred from scope (already in the planning log)

* WI-01 (medium) — ingress-level protection for admin writes (durable replacement for the removed app-code gate).
* WI-04 (low) — scrub pre-existing `#35x` task tokens from `admin.py` per-route docstrings.
* WI-06 (low) — back-fill the missing ADR 0030 index row in `adr/README.md`.
* WI-07 (low) — optionally reclassify the BUG-0090 registry `Area` `infra` → `backend`.

## Overall Status

**✅ Complete (delivered scope) — no Critical or Major findings.** All plan items for Phases 1–5 and 6.1–6.2 are verified against source; the full suite is green; the fix is correct and faithful to the user directive and MACAE. The only outstanding items are (a) the intentionally-gated live deploy (Step 6.3 → flip BUG-0090 to `fixed`), and (b) three Minor follow-ups (chiefly the stale live docs, F-1). No rework of the shipped code is required.

**Reviewer note:** Before the backend is exposed publicly, close the accepted security risk (WI-01) at the ingress layer — the app no longer gates admin routes by design.
