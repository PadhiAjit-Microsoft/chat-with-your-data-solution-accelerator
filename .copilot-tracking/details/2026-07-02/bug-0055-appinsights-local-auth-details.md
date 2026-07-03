<!-- markdownlint-disable-file -->
# Implementation Details: BUG-0055 — enable App Insights local auth (match MACAE)

## Context Reference

Sources:
* .copilot-tracking/research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md (selected approach: match MACAE)
* .copilot-tracking/research/subagents/2026-07-02/bug-0055-infra-test-and-adr-impact-research.md (test + ADR impact)
* .copilot-tracking/research/subagents/2026-07-02/bug-0055-macae-disablelocalauth-verification.md (MACAE omits disableLocalAuth on App Insights)

Selected approach: set the App Insights component `disableLocalAuth: false` so the existing connection-string-only `configure_azure_monitor` calls ingest via instrumentation key — identical to MACAE. No application code change. Retain the `Monitoring Metrics Publisher` UAMI role (harmless, reversible, avoids test churn). Reverses ADR-0018 Decision #2 → requires an ADR amendment.

## Implementation Phase 1: Bicep — enable App Insights local auth

<!-- parallelizable: false -->

### Step 1.1: Add a test asserting App Insights local auth is enabled

Test-first (Hard Rule #2). The infra harness `v2/tests/infra/test_main_bicep.py` is a pure Python string-parser: it reads `main.bicep`, slices the `applicationInsights` module block, and asserts substrings. Add a test that asserts the App Insights slice contains `disableLocalAuth: false` (and does NOT contain `disableLocalAuth: true`). This test fails against the current `true` literal, then passes after Step 1.2.

Files:
* v2/tests/infra/test_main_bicep.py - add `test_application_insights_enables_local_auth` near the existing `test_application_insights_grants_metrics_publisher_to_uami` (~line 347); reuse the same App Insights module-slice fixture/helper (`_slice_module` bracketing `"module applicationInsights "` … `"// Virtual network "`).

Discrepancy references:
* Addresses DR-01 (test-first anchor for an infra-only change).

Success criteria:
* New test `test_application_insights_enables_local_auth` exists and initially FAILS against the current `disableLocalAuth: true`.
* Test asserts `"disableLocalAuth: false"` is in the App Insights slice.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0055-infra-test-and-adr-impact-research.md (harness is string-parse; slice bracketing pattern)

Dependencies:
* None.

### Step 1.2: Flip `disableLocalAuth: true` → `false` on the App Insights module

Change the single inline literal on the AVM `applicationInsights` module. Keep the `Monitoring Metrics Publisher` role assignment (lines ~330-336) unchanged — it becomes unused but is harmless and preserves a clean revert path back to Entra-only ingestion.

Files:
* v2/infra/main.bicep - line 326: `disableLocalAuth: true` → `disableLocalAuth: false`. Update the adjacent comment (if it justifies `true`) to state local auth is enabled to match MACAE's `avm/res/insights/component` (which omits the flag) for connection-string/ikey ingestion; note the retained role is reserved for a future revert to Entra-only ingestion.

**Guardrail (DR-03):** edit **line 326 only** — the App Insights module literal. Do NOT do a global find/replace. The sibling `disableLocalAuth: true` literals on aiServices / aiSearch / storage (~L521/723/798/861) MUST stay `true`; they are different data planes and out of scope. The Step 1.1 guard is App Insights-slice-scoped and cannot catch a sibling regression, so apply the single-line edit manually and confirm via `grep -n "disableLocalAuth" infra/main.bicep` that exactly one literal reads `false` and all others remain `true`.

Discrepancy references:
* Addresses DD-01 (this reverses the earlier credential-based recommendation per user decision).

Success criteria:
* `main.bicep` App Insights module has `disableLocalAuth: false`.
* The `Monitoring Metrics Publisher` role assignment is retained (no removal).
* `test_application_insights_enables_local_auth` now PASSES; `test_application_insights_grants_metrics_publisher_to_uami` still PASSES (role retained).

Context references:
* .copilot-tracking/research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md (Complete Examples — the one-line Bicep change)

Dependencies:
* Step 1.1 (test exists and fails first).

### Step 1.3: Update stale assertion-message prose referencing `disableLocalAuth=true`

The role test's failure message at `v2/tests/infra/test_main_bicep.py` line 353 reads "…with disableLocalAuth=true…". Since local auth is now enabled, update that prose so the message explains the role is retained for a potential future revert to Entra-only ingestion (not that it is currently required). Message-only change; the three role assertions stay.

Files:
* v2/tests/infra/test_main_bicep.py - line ~353: update the failure-message string only.

Success criteria:
* No test assertion logic changes; the message no longer claims `disableLocalAuth=true`.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0055-infra-test-and-adr-impact-research.md (line 353 prose is stale after the flip)

Dependencies:
* Step 1.2.

### Step 1.4: Validate phase changes

Run the infra harness + a Bicep build to confirm the template still compiles.

Validation commands:
* `v2\.venv\Scripts\python.exe -m pytest tests/infra/test_main_bicep.py -v` (run from `v2`) - the App Insights tests pass.
* `az bicep build --file infra/main.bicep --stdout > $null` (run from `v2`) - template compiles with no errors.

Success criteria:
* All `test_main_bicep.py` tests green.
* `az bicep build` exits 0 (no compile/lint errors).

Dependencies:
* Steps 1.1-1.3.

## Implementation Phase 2: ADR-0018 amendment

<!-- parallelizable: true -->

### Step 2.1: Record the reversal as ADR-0018 Amendment 2

The change reverses ADR-0018 Decision #2 (Entra-only ingestion), its "UAMI + Entra-only ingestion stays" positive consequence, and rejected-Alternative #3. Add an amendment matching the existing heading style, placed after "## Amendment 1" and before "## References".

Files:
* v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md - add `## Amendment 2 (2026-07-02) — enable App Insights local auth to match MACAE (BUG-0055)`. Content: (a) what changed (`disableLocalAuth: true` → `false` on the App Insights component; `Monitoring Metrics Publisher` role retained but now unused); (b) why (match MACAE's proven pattern; the app is already connection-string-only so this restores ingestion with zero app-code change; smallest possible change); (c) tradeoff (ikey-based ingestion is a weaker bar than Entra-token ingestion — accepted because MACAE, a shipped Microsoft accelerator, uses this posture); (d) revert path (flip back to `true` + pass a sync `ManagedIdentityCredential` per the research doc's rejected alternative; the retained role makes this a one-line revert).

Discrepancy references:
* Addresses DD-01 (documents the ADR reversal the user's decision requires).

Success criteria:
* ADR-0018 has an "## Amendment 2" section matching the Amendment 1 heading style.
* The amendment names the retained role and the documented revert path.

Context references:
* .copilot-tracking/research/subagents/2026-07-02/bug-0055-infra-test-and-adr-impact-research.md (Amendment 1 heading pattern; which decisions the change reverses)

Dependencies:
* None (doc-only; independent of Phase 1 files).

## Implementation Phase 3: Deploy, live-verify, close bug

<!-- parallelizable: false -->

### Step 3.1: Provision to apply the Bicep change

`disableLocalAuth` is a template property, applied by `azd provision` (not `azd deploy`). This provision also applies the still-pending backend `AZURE_APP_INSIGHTS_CONNECTION_STRING` env-var rename (the secondary deploy-state issue) in the same pass.

Commands (run from a terminal already at `v2`):
* `azd provision` (or `azd up`) - re-applies the App Insights `disableLocalAuth: false` + backend env-var rename. Set `$env:AZURE_CORE_ONLY_SHOW_ERRORS="true"` to quiet az warnings. Do NOT run azd from the repo root (root has the v1 `azure.yaml`).

Success criteria:
* `azd provision` completes without error.
* `az containerapp show -g <RESOURCE_GROUP> -n ca-backend-<SUFFIX> --query "properties.template.containers[0].env[?name=='AZURE_APP_INSIGHTS_CONNECTION_STRING']"` returns a non-empty value.

Dependencies:
* Phase 1 complete (Bicep valid).

### Step 3.2: Live-verify telemetry from both runtimes

Generate a little traffic (a chat request to the backend; trigger an ingestion on the function), wait ~1-3 min, then query App Insights.

KQL (classic AI Logs blade):
```kusto
union isfuzzy=true requests, traces, dependencies, exceptions, customMetrics
| where timestamp > ago(30m)
| summarize items = count(), LastSeen = max(timestamp) by itemType
| order by LastSeen desc
```

Success criteria:
* Non-zero `traces`/`requests`/`dependencies` from the backend AND function within ~3 min (previously `[0, null, null]`).

Dependencies:
* Step 3.1.

### Step 3.3: Mark BUG-0055 fixed + worklog

Files:
* v2/docs/bugs.md - registry row line 114: set Fixed date `2026-07-02`, Status `fixed`; append a resolution to the detail section (line ~998) describing the MACAE-match (`disableLocalAuth: false`, connection-string ingestion, no credential, ADR-0018 Amendment 2) and the live-verified KQL result.
* v2/docs/worklog/2026-07-02.md - add a BUG-0055 entry.

Success criteria:
* BUG-0055 row shows `fixed` (2026-07-02); detail cites Amendment 2 + live verification.
* Open-bug count drops to 3 (BUG-0081, BUG-0082, BUG-0090).
* Env-ID gate `v2/tests/shared/test_no_env_specific_content.py` passes (placeholders only).

Dependencies:
* Step 3.2 (live verification confirms the fix before marking fixed).

## Dependencies

* Azure CLI + azd authenticated to the target subscription; `v2/.azure/<AZD_ENV_NAME>/.env` present.
* `v2\.venv` Python environment (`uv sync`).
* `az bicep` available.

## Success Criteria

* App Insights `disableLocalAuth: false` in `main.bicep`; infra tests green; `az bicep build` clean.
* ADR-0018 Amendment 2 records the reversal + revert path.
* Live App Insights shows non-zero telemetry from backend + function.
* BUG-0055 marked fixed; no env-specific IDs in any tracked file.
