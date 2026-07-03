---
applyTo: '.copilot-tracking/changes/2026-07-02/bug-0055-appinsights-local-auth-changes.md'
---
<!-- markdownlint-disable-file -->
# Implementation Plan: BUG-0055 — enable App Insights local auth (match MACAE)

## Overview

Fix BUG-0055 (App Insights receives zero telemetry) by setting the App Insights component `disableLocalAuth: false` so the existing connection-string-only telemetry ingests via instrumentation key — matching MACAE — with no application code change.

## Objectives

### User Requirements

* Refactor the telemetry fix to match MACAE's behavior. — Source: user message 2026-07-02 ("I want refactor to match macae behavior").
* Confirm the approach uses connection-string ingestion as MACAE does, not the credential/token path. — Source: user messages 2026-07-02 ("use credentials a no toke", "check what MACAE does", "does this variable exist in MACAE disableLocalAuth: true?").

### Derived Objectives

* Reverse ADR-0018's `disableLocalAuth: true` decision via a formal amendment. — Derived from: the change reverses a documented ADR decision (Hard Rule #0/#10; ADR-0018 Decision #2).
* Preserve a clean revert path to Entra-only ingestion by retaining the `Monitoring Metrics Publisher` UAMI role. — Derived from: avoiding role-test churn and keeping the credential alternative one flip away.
* Land the infra change test-first against the existing string-parse harness. — Derived from: Hard Rule #2 (test-first).

## Context Summary

### Project Files

* v2/infra/main.bicep - App Insights AVM module; `disableLocalAuth: true` inline literal at line 326; `Monitoring Metrics Publisher` role assignment lines ~330-336.
* v2/tests/infra/test_main_bicep.py - pure Python string-parse harness; `test_application_insights_grants_metrics_publisher_to_uami` (~L347); stale message at L353.
* v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md - decision record to amend; `## Amendment 1 (2026-06-23) — …` heading pattern.
* v2/docs/bugs.md - BUG-0055 registry row line 114; detail section line ~998.
* v2/docs/worklog/2026-07-02.md - today's worklog.

### References

* .copilot-tracking/research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md - root cause + selected match-MACAE approach.
* .copilot-tracking/research/subagents/2026-07-02/bug-0055-infra-test-and-adr-impact-research.md - test + ADR impact.
* .copilot-tracking/research/subagents/2026-07-02/bug-0055-macae-disablelocalauth-verification.md - MACAE omits `disableLocalAuth` on App Insights.

### Standards References

* .github/copilot-instructions.md - Hard Rule #1 (one unit/turn), #2 (test-first), #10 (structural confirmation), #18 (no env-specific IDs), #19 (bugs.md + worklog tracking).
* .github/instructions/v2-infra.instructions.md - Bicep + azd conventions.

## Implementation Checklist

### [x] Implementation Phase 1: Bicep — enable App Insights local auth

<!-- parallelizable: false -->

* [x] Step 1.1: Add a test asserting App Insights local auth is enabled
  * Details: .copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md (Lines 17-36)
* [x] Step 1.2: Flip `disableLocalAuth: true` → `false` on the App Insights module
  * Details: .copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md (Lines 37-59)
* [x] Step 1.3: Update stale assertion-message prose referencing `disableLocalAuth=true`
  * Details: .copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md (Lines 60-75)
* [x] Step 1.4: Validate phase changes (pytest infra harness + `az bicep build`)
  * Details: .copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md (Lines 76-90)

### [x] Implementation Phase 2: ADR-0018 amendment

<!-- parallelizable: true -->

* [x] Step 2.1: Record the reversal as ADR-0018 Amendment 2
  * Details: .copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md (Lines 95-114)

### [x] Implementation Phase 3: Deploy, live-verify, close bug

<!-- parallelizable: false -->

* [x] Step 3.1: Provision to apply the Bicep change (`azd provision` from v2)
  * Details: .copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md (Lines 119-132)
* [x] Step 3.2: Live-verify telemetry from both runtimes (KQL union query)
  * Details: .copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md (Lines 133-150)
* [x] Step 3.3: Mark BUG-0055 fixed + worklog entry
  * Details: .copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md (Lines 151-164)

### [x] Implementation Phase 4: Validation

<!-- parallelizable: false -->

* [ ] Step 4.1: Run full infra + shared gate validation
  * `v2\.venv\Scripts\python.exe -m pytest tests/infra/test_main_bicep.py tests/shared/test_no_env_specific_content.py -v`
  * `az bicep build --file infra/main.bicep --stdout > $null`
* [ ] Step 4.2: Fix minor validation issues
  * Iterate on any string-parse assertion mismatch or bicep lint warning
* [ ] Step 4.3: Report blocking issues
  * If `azd provision` fails or telemetry stays zero after ~3 min, document and recommend re-research (do not brute-force)

## Planning Log

See .copilot-tracking/plans/logs/2026-07-02/bug-0055-appinsights-local-auth-log.md for discrepancy tracking, implementation paths considered, and suggested follow-on work.

## Dependencies

* Azure CLI + azd authenticated to the target subscription; `v2/.azure/<AZD_ENV_NAME>/.env` present.
* `v2\.venv` Python environment (`uv sync`).
* `az bicep` available.

## Success Criteria

* App Insights `disableLocalAuth: false` in `main.bicep`; `test_main_bicep.py` green; `az bicep build` clean. — Traces to: research selected approach; test-first (Hard Rule #2).
* ADR-0018 Amendment 2 records the reversal and the revert path. — Traces to: Derived Objective (ADR reversal).
* Live App Insights shows non-zero telemetry from backend + function. — Traces to: BUG-0055 symptom (`[0, null, null]`).
* BUG-0055 marked fixed; no env-specific IDs in any tracked file. — Traces to: Hard Rule #18/#19.
