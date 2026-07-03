<!-- markdownlint-disable-file -->
# Task Review: BUG-0055 — enable App Insights local auth (match MACAE)

## Review Metadata

* **Review date**: 2026-07-02
* **Plan**: .copilot-tracking/plans/2026-07-02/bug-0055-appinsights-local-auth-plan.instructions.md
* **Changes log**: .copilot-tracking/changes/2026-07-02/bug-0055-appinsights-local-auth-changes.md
* **Research**: .copilot-tracking/research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md
* **Planning log**: .copilot-tracking/plans/logs/2026-07-02/bug-0055-appinsights-local-auth-log.md
* **Reviewer**: Task Reviewer

## Scope

Implemented and under review: Phase 1 (Bicep local-auth flip + infra tests) and Phase 2 (ADR-0018 Amendment 2). Phase 3 (live `azd provision` + telemetry verification + bug close) and Phase 4 (final validation) are **intentionally deferred** pending user go-ahead for the shared-infra deploy — not defects.

## Summary of Findings

Phases 1 and 2 (the implemented scope) are **verified** against the plan, details, and research. Both guardrails hold (single `disableLocalAuth: false` on the App Insights module; four sibling literals stay `true`), the change is test-first, and the ADR amendment is complete and research-consistent. The single minor finding (M-1, an ADR doc-accuracy nuance) was **resolved during this review**.

| Severity | Count | Notes |
|----------|-------|-------|
| Critical | 0 | — |
| Major    | 0 | — |
| Minor    | 1 | M-1 (ADR Alternative #3 mischaracterization) — **resolved during review** |
| Info     | 5 | Non-issues (sibling-name label, combined test count, permitted test-file cross-refs, cross-amendment continuity, MACAE fact match) |

## RPI Validation (per phase)

### Phase 1 — Bicep local-auth flip + infra tests — ✅ Verified

* Validation doc: .copilot-tracking/reviews/rpi/2026-07-02/bug-0055-appinsights-local-auth-plan-001-validation.md
* Findings: 0 critical / 0 major / 0 minor / 3 info.
* Evidence: whole-file `disableLocalAuth` scan — **line 326 = `false`** (App Insights, the intended flip); siblings `aiServices` (L522), `speechService` (L724), `cogContentSafety` (L799), `aiSearch` (L862) all still `true`; Cosmos `disableLocalAuthentication` (L1308) untouched. `Monitoring Metrics Publisher` role retained (L328-335). New `test_application_insights_enables_local_auth` asserts both positive + negative. No `v2/src` code touched; no env-ID leak. `pytest tests/infra/test_main_bicep.py` → **38 passed**; `az bicep build` → exit 0.

### Phase 2 — ADR-0018 Amendment 2 — ✅ Verified

* Validation doc: .copilot-tracking/reviews/rpi/2026-07-02/bug-0055-appinsights-local-auth-plan-002-validation.md
* Findings: 0 critical / 0 major / 1 minor (M-1) / 2 info.
* Evidence: `## Amendment 2` at ADR line 102, correctly placed after Amendment 1 (L74) and before References (L148), heading + prose style matched. All four content elements (a) what changed, (b) why/MACAE, (c) tradeoff + named reversed ADR elements, (d) revert path with both call sites + sync credential — present and accurate. No env-ID leak. Primary narrative is the SELECTED match-MACAE approach; credential path appears only as the labeled revert alternative.
* M-1 (minor): Amendment 2 claimed it reverses rejected Alternative #3 "which had considered a posture with local auth left enabled" — but the ADR's Alternative #3 is actually "skip the `Monitoring Metrics Publisher` role" (retaining `disableLocalAuth: true`). **Resolved during review**: the parenthetical was corrected to state the ADR never enumerated a "leave local auth enabled" alternative and that Alternative #3's drop-the-role stance is untouched (the role is retained).

## Implementation Quality

Direct quality assessment (the Implementation Validator subagent was blocked by a tooling limitation; the reviewer read the changed regions directly):

* **Bicep** (main.bicep L326-338): clean boolean flip; comment is present-tense, accurate, and explains both the local-auth rationale (match MACAE) and why the role is retained-but-unused. No stale/process narrative. camelCase conventions intact; no duplicated literals introduced. **Pass.**
* **Tests** (test_main_bicep.py): `test_application_insights_enables_local_auth` is well-named, focused, asserts positive + negative with clear messages; the module-level comment block was refreshed to the enabled-local-auth reality; the metrics-publisher test's three role assertions are intact. **Pass.**
* **ADR** (Amendment 2): well-structured, consistent story across Bicep comment + test comments + ADR (local auth on, role retained-but-unused, MACAE match, one-flip revert). After the M-1 fix, factually accurate. **Pass.**
* Naming stability (Hard Rule #11): no casual renames; new test follows `test_*` convention. Env-ID hygiene (Hard Rule #18): clean. No dead code / leftover TODO.

**Quality verdict: Pass** (M-1 resolved).

## Validation Command Outputs

| Command | Result |
|---------|--------|
| `pytest tests/infra/test_main_bicep.py tests/shared/test_no_env_specific_content.py` | ✅ 43 passed (38 infra + 5 env-ID gate) |
| `pytest tests/infra/test_main_bicep.py` (RPI re-run) | ✅ 38 passed |
| `az bicep build --file infra/main.bicep` | ✅ exit 0 |

## Missing Work and Deviations

* **Phase 3 — deploy, live-verify, close bug — NOT executed (deferred, not a defect).** Requires a live `azd provision` to shared infrastructure; paused for user go-ahead. Until run: BUG-0055 stays `open` in bugs.md, and the fix is not yet cloud-verified.
* **Phase 4 — final validation — partially satisfied.** The local infra + env-ID gate run (43 passed) covers Step 4.1's intent; the full post-provision sign-off belongs with Phase 3.
* No unplanned deviations. One in-scope cleanup beyond the plan: the stale module-level test comment refresh (Phase 1 implementer's suggested step) — recorded in the changes log.

## Follow-Up Work

### Deferred from scope (execute when authorized)

* Run Phase 3: `azd provision` (from `v2`) → generate traffic → union KQL telemetry check → mark BUG-0055 `fixed` in bugs.md + worklog (open bugs → 3). Owner: user go-ahead required (shared-infra deploy).

### Discovered during review

* (Resolved) M-1 ADR Alternative #3 characterization — fixed during review; no further action.
* WI-01 (from planning log): remove the temporary self-granted verification RBAC roles after live verification. (medium)

## Overall Status

**✅ Complete (implemented scope: Phases 1-2).** No critical or major findings; the one minor (M-1) was resolved during review. The Bicep flip, test-first guard, retained role, and ADR-0018 Amendment 2 are verified and validation-green. **Phase 3 (live `azd provision` + telemetry verification + bug close) remains intentionally pending user authorization** — the fix is code-complete and locally validated but not yet cloud-verified.

## Re-Review (2026-07-02)

Second review pass covering the delta since the first review (the M-1 ADR fix + changes-log update).

* **M-1 verified resolved.** Re-read ADR-0018 Amendment 2 "Tradeoff" paragraph (v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md L131-140). It now correctly (a) reverses Decision #2 + the *Positive* consequence, (b) states the ADR never enumerated a "leave local auth enabled" alternative and introduces that posture deliberately, and (c) clarifies the `Monitoring Metrics Publisher` role is **retained, not reversed**, with rejected Alternative #3's "drop the role" stance untouched. Factually accurate against the ADR's actual Alternative #3 ("skip the role assignment"). The prior mischaracterization is gone.
* **Changes log** records the M-1 correction under the ADR Modified entry.
* **Validation re-run:** `pytest tests/infra/test_main_bicep.py tests/shared/test_no_env_specific_content.py` → **43 passed**. No regression from the doc-only edit.
* **New findings this pass:** 0 (critical/major/minor). No open findings remain.
* **Status unchanged:** ✅ Complete for the implemented scope (Phases 1-2). Phase 3 (live deploy + telemetry verify + bug close) still pending user authorization; BUG-0055 stays `open` in bugs.md until cloud-verified.
