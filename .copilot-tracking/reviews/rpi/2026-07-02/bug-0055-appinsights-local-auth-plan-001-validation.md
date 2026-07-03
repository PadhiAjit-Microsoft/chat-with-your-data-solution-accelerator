<!-- markdownlint-disable-file -->
# RPI Validation: BUG-0055 — Phase 1 (Bicep — enable App Insights local auth)

**Validation date:** 2026-07-02
**Mode:** RPI Validator (read-only; no source modified)
**Phase under validation:** Phase 1 — "Bicep — enable App Insights local auth" (Steps 1.1–1.4)
**Phase status:** ✅ **Verified**

## Inputs

* Plan: `.copilot-tracking/plans/2026-07-02/bug-0055-appinsights-local-auth-plan.instructions.md`
* Changes log: `.copilot-tracking/changes/2026-07-02/bug-0055-appinsights-local-auth-changes.md`
* Details: `.copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md`
* Research: `.copilot-tracking/research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md`

## Executive Summary

Phase 1 is fully implemented and matches the plan, details, and research. The
single App Insights literal was flipped `true → false` at `v2/infra/main.bicep`
line 326; all four sibling `disableLocalAuth: true` literals remain `true`; the
Cosmos `disableLocalAuthentication` property (distinct name) is untouched. The
`Monitoring Metrics Publisher` role is retained. The test-first assertion
(`test_application_insights_enables_local_auth`) exists with both required
sub-assertions; the role test's three assertions are intact with only prose
updated. Validation gates are green: `pytest tests/infra/test_main_bicep.py` →
38 passed; `az bicep build` → exit 0. No application code was touched; no
env-specific IDs were introduced.

**Findings by severity:** Critical 0 · Major 0 · Minor 0 · Info 3

## Per-Step Verification

| Step | Requirement | Evidence | Status |
|------|-------------|----------|--------|
| 1.1 | Add `test_application_insights_enables_local_auth` asserting `disableLocalAuth: false` present + `disableLocalAuth: true` absent (test-first) | `v2/tests/infra/test_main_bicep.py` L368–L387: test present with both asserts against `application_insights_slice`. Changes log confirms failed pre-flip / green after (test-first). | ✅ Verified |
| 1.2 | Flip line 326 `true → false` on App Insights module ONLY; siblings stay `true`; retain `Monitoring Metrics Publisher`; present-tense comment | `main.bicep` L326 = `disableLocalAuth: false`; siblings L522/724/799/862 = `true`; role block L329–L335 retained; comment L327–L330 present-tense, no process narrative | ✅ Verified |
| 1.3 | Update stale `disableLocalAuth=true` failure-message prose on `test_application_insights_grants_metrics_publisher_to_uami`; keep 3 role assertions | Message updated to "…retained for a potential future revert…(local auth is currently enabled)"; assertions `roleAssignments:`, `Monitoring Metrics Publisher`, `userAssignedIdentity.outputs.principalId` all intact (diff shows no logic change) | ✅ Verified |
| 1.4 | pytest infra harness green; `az bicep build` exit 0 | `pytest tests/infra/test_main_bicep.py -q` → **38 passed** in 0.04s; `az bicep build --file infra/main.bicep` → **exit 0** | ✅ Verified |

## `disableLocalAuth` Occurrence List (whole-file scan of `v2/infra/main.bicep`)

Full grep of `disableLocalAuth` returned 8 matches. Classified below (literal
vs. comment vs. distinct property), with the owning module verified by reading
each region.

| Line | Text | Owning module | Value | Verdict |
|------|------|---------------|-------|---------|
| 326 | `disableLocalAuth: false` | `applicationInsights` (App Insights `avm/res/insights/component`) | **false** | ✅ Intended flip target — the ONE literal that reads `false` |
| 522 | `disableLocalAuth: true` | `aiServices` (`cognitive-services/account`, kind `AIServices`) | true | ✅ Sibling retained |
| 724 | `disableLocalAuth: true` | `speechService` (`cognitive-services/account`, kind `SpeechServices`) | true | ✅ Sibling retained |
| 799 | `disableLocalAuth: true` | `cogContentSafety` (`cognitive-services/account`, kind `ContentSafety`) | true | ✅ Sibling retained |
| 862 | `disableLocalAuth: true` | `aiSearch` (`search/search-service`) | true | ✅ Sibling retained |
| 903 | comment (`…disableLocalAuth is on…`) | aiSearch role comment | n/a | Comment, not a literal |
| 1293 | comment (`…AAD-only (disableLocalAuth)…`) | cosmosDb comment | n/a | Comment, not a literal |
| 1308 | `disableLocalAuthentication: true` | `cosmosDb` (`document-db/database-account`) | true | ✅ **Distinct property** (`disableLocalAuthentication`, not `disableLocalAuth`) — correctly retained |

**Guardrail result (CRITICAL check):** EXACTLY ONE `disableLocalAuth` literal
reads `false` (line 326 = App Insights). All four sibling `disableLocalAuth`
literals read `true`. The Cosmos `disableLocalAuthentication` is a separate
property and remains `true`. **No sibling regression — DR-03 guardrail
satisfied.**

> Note on the plan's guardrail naming: the details doc named the siblings as
> "aiServices / aiSearch / storage (~L521/723/798/861)". The actual siblings
> are **aiServices, speechService, cogContentSafety, aiSearch** — there is no
> `storage` module carrying a `disableLocalAuth` literal (storage governs data-
> plane auth differently), and Cosmos uses `disableLocalAuthentication`. The
> line estimates are accurate (off-by-one). See Info-1.

## Role Assignment Retention (MAJOR check)

`v2/infra/main.bicep` L328–L335 — the `roleAssignments` block granting
`Monitoring Metrics Publisher` to `userAssignedIdentity.outputs.principalId` is
**present and unchanged**. The bicep diff shows only the literal + the comment
above it changed; the role block was not removed. ✅ Retained.

## Adjacent Comment (Hard Rule #16 spirit)

`main.bicep` L327–L330, present-tense, no dates / unit IDs / process narrative:

> Local auth is enabled so connection-string / instrumentation-key
> ingestion is accepted, matching MACAE's `avm/res/insights/component`
> (which omits the flag). The `Monitoring Metrics Publisher` role below
> is retained but unused, reserved for a revert to Entra-only ingestion.

✅ Clean.

## Test File Details (`v2/tests/infra/test_main_bicep.py`)

* `test_application_insights_enables_local_auth` (L368–L387): asserts
  `"disableLocalAuth: false" in application_insights_slice` AND
  `"disableLocalAuth: true" not in application_insights_slice`. ✅
* `test_application_insights_grants_metrics_publisher_to_uami` (L347–L367):
  three role assertions intact — `roleAssignments:`,
  `_MONITORING_METRICS_PUBLISHER_ROLE_NAME` (= "Monitoring Metrics Publisher"),
  `userAssignedIdentity.outputs.principalId`. Message no longer claims
  `disableLocalAuth=true` (now "local auth is currently enabled"). ✅
* Module-level comment block (~L316–L343): refreshed from the prior
  `disableLocalAuth: true` narration to the enabled-local-auth reality + the
  RBAC-free revert path; also dropped a brittle "~line 552" reference. ✅

## Validation Command Results

| Command (run from `v2`) | Result |
|--------------------------|--------|
| `.venv\Scripts\python.exe -m pytest tests/infra/test_main_bicep.py -q` | **38 passed** in 0.04s |
| `az bicep build --file infra/main.bicep --stdout > $null` | **exit 0** |

> The changes log records "43 passed" for the combined run
> (`test_main_bicep.py` + `tests/shared/test_no_env_specific_content.py`);
> the scoped run here (test_main_bicep only) reports 38, consistent with the
> combined figure (38 + 5). See Info-2.

## Scope & Env-ID Checks

* `git status --short` over `v2/infra/main.bicep`, `v2/tests/infra/test_main_bicep.py`,
  `v2/src` → only the two expected files modified; **no `v2/src` (backend /
  functions) application code touched**. ✅
* Bicep + test diffs contain no subscription / tenant / UAMI / resource-group /
  suffix values — only a boolean flip, English comments, and Python asserts.
  **Hard Rule #18 satisfied.** ✅

## Findings

### Info

* **Info-1 — Guardrail sibling naming imprecise (no functional impact).** The
  details doc labels the siblings "aiServices / aiSearch / storage"; the real
  set is aiServices, speechService, cogContentSafety, aiSearch (no storage
  literal; Cosmos uses `disableLocalAuthentication`). All four correct siblings
  stayed `true`, so the intent (no sibling regression) held. Documentation-only
  nuance for future edits.
* **Info-2 — pytest count differs from changes log by design.** Changes log:
  "43 passed" (combined with the env-ID gate). This scoped validation: 38
  passed (`test_main_bicep.py` only). Consistent; no discrepancy.
* **Info-3 — Test-file comments cite Amendment 2 / BUG-0055 / MACAE.** Permitted
  — Hard Rule #16 (no process narrative) is scoped to `v2/src/**` production
  code; test files are outside the gate, and ADR / BUG identifiers are allowed
  references. No violation.

### Minor / Major / Critical

None.

## Coverage Assessment

Phase 1 coverage is **complete**. Every Step 1.1–1.4 requirement in the plan,
details, and research has a corresponding, verified change. The one CRITICAL
guardrail (no sibling `disableLocalAuth` regression) and the one MAJOR guardrail
(retain the `Monitoring Metrics Publisher` role) both hold. Both validation
gates are green.

## Clarifying Questions

None — all Phase 1 items resolved from available context.

## Recommended Next Validations (not performed this session)

- [ ] Validate Phase 2 (ADR-0018 Amendment 2) — confirm the amendment heading
      style, the reversed-decision naming, and the credential-based revert path
      in `v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md`.
- [ ] Run the full shared gate suite (`tests/shared/test_no_env_specific_content.py`
      and the other shared AST gates) once more before any provision.
- [ ] Phase 3 is deployment/live-verify (intentionally not executed) — validate
      after `azd provision` via the union KQL when the user authorizes rollout.
