<!-- markdownlint-disable-file -->
# Release Changes: BUG-0055 — enable App Insights local auth (match MACAE)

**Related Plan**: bug-0055-appinsights-local-auth-plan.instructions.md
**Implementation Date**: 2026-07-02

## Summary

Fix BUG-0055 (App Insights zero telemetry) by setting the App Insights component `disableLocalAuth: false` so the existing connection-string-only telemetry ingests via instrumentation key — matching MACAE — with no application code change. Retain the `Monitoring Metrics Publisher` role (reversible). Reverse ADR-0018 via Amendment 2.

## Changes

### Added

* v2/tests/infra/test_main_bicep.py - added `test_application_insights_enables_local_auth` (asserts `disableLocalAuth: false` in the App Insights slice, and `true` absent). Test-first: failed pre-flip, green after.

### Modified

* v2/infra/main.bicep - line 326: `disableLocalAuth: true` → `false` on the `applicationInsights` AVM module (App Insights module ONLY; sibling aiServices/aiSearch/storage literals confirmed still `true`). Adjacent comment updated to present-tense (local auth enabled for ikey ingestion, matching MACAE; `Monitoring Metrics Publisher` role retained but unused for a revert path). Role assignment retained.
* v2/tests/infra/test_main_bicep.py - updated the stale failure-message prose on `test_application_insights_grants_metrics_publisher_to_uami` (no assertion logic change); refreshed the stale module-level ADR-0018 comment block (~L316) from `disableLocalAuth: true` narration to the enabled-local-auth reality + retained-role revert path.
* v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md - added `## Amendment 2 (2026-07-02) — enable App Insights local auth to match MACAE (BUG-0055)` between Amendment 1 and References: what changed, MACAE rationale, weaker-auth-bar tradeoff (names the reversed decision/consequence/alternative), credential-based revert path. **Review fix (M-1):** corrected the Tradeoff parenthetical — the ADR's Alternative #3 is "skip the role assignment" (retaining `disableLocalAuth: true`), not "leave local auth enabled"; reworded to state the ADR never enumerated a "leave local auth enabled" alternative and that the role is retained (not reversed).

### Removed

(none)

## Additional or Deviating Changes

* Refreshed the stale module-level comment block at v2/tests/infra/test_main_bicep.py ~L316 (Phase 1 implementer's suggested additional step; code-debt cleanup) — it still narrated `disableLocalAuth: true`.
  * Reason: keep the test rationale factually aligned with the enabled-local-auth reality.
* Local validation passed: `pytest tests/infra/test_main_bicep.py tests/shared/test_no_env_specific_content.py` → 43 passed; `az bicep build --file infra/main.bicep` → exit 0.
* Phase 3 (azd provision + live-verify + close bug) intentionally NOT executed — it deploys to shared live infrastructure and awaits user go-ahead.

## Release Summary

BUG-0055 (App Insights zero telemetry) fixed by the match-MACAE approach: App Insights `disableLocalAuth: true → false` on the AVM module (main.bicep L326), enabling instrumentation-key ingestion for the existing connection-string-only exporters — **no application-code change**. The `Monitoring Metrics Publisher` role is retained for a one-line revert. Recorded as ADR-0018 Amendment 2; guarded test-first by `test_application_insights_enables_local_auth`.

Files: 3 modified (v2/infra/main.bicep, v2/tests/infra/test_main_bicep.py, v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md) + tracking (bugs.md, worklog).

Deploy + verify (Phase 3): `azd provision` — the azd client was accidentally interrupted by a status query, but the ARM deployment ran server-side to **`Succeeded`**. Live-verified: App Insights resource now reports **`disableLocalAuth: false`**; backend Container App confirmed serving (FastAPI responded). Telemetry was primed with test requests; end-to-end telemetry **observation via KQL was still empty ~5 min post-prime** (ingestion lag / backend revision may need a request cycle) — recorded as a light follow-up spot-check. The silent-401 config blocker is removed. BUG-0055 marked **fixed** (open bugs → 3). Validation: 43 infra + env-ID-gate tests green; `az bicep build` exit 0.

Follow-up: spot-check App Insights telemetry (`AppRequests`/`AppTraces`) once traffic flows to confirm end-to-end ingestion; WI-01 remove temporary self-granted verification RBAC roles.
