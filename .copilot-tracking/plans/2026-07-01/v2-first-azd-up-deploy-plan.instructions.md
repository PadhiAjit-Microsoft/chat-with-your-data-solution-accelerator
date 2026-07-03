---
applyTo: '.copilot-tracking/changes/2026-07-01/v2-first-azd-up-deploy-changes.md'
---
<!-- markdownlint-disable-file -->
# Implementation Plan: CWYD v2 — First `azd up` Deploy Path

## Overview

Take CWYD v2 from green-but-undeployed to a first successful `azd up` in the new tenant: unblock the one hard frontend build blocker (WI-07), run read-only pre-deploy verification, provision + deploy all three container services, then validate the live deployment grounds on seeded sample data.

## Objectives

### User Requirements

* Plan the full path to a first `azd up` deployment (unblock + verify + deploy + validate) — Source: user selection 2026-07-01 (Plan scope → Option A).
* New tenant confirmed to have sufficient deploy access — Source: user statement 2026-07-01.

### Derived Objectives

* Resolve WI-07 (`TS6133` in `Configuration.tsx`) so the frontend production container image builds — Derived from: research (only hard code blocker to `azd up`).
* Verify `gpt-5.1` + `text-embedding-3-large` quota before provision (fresh tenant) — Derived from: WI-01 / DR-08.
* Validate post-deploy grounding on the seeded benefits documents — Derived from: resolved WI-03 (postdeploy sample-data hook exists).

## Context Summary

### Project Files

* v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx - WI-07 fix site (uncomment audit render at ~1040-1045; `formatActor` at 628).
* v2/src/frontend/src/models/admin.tsx - `RuntimeConfig.updated_by` field (line 278) the render reads; already present.
* v2/tests/frontend/pages/admin/Configuration/Configuration.test.tsx - coupled failing vitest (line 505) that turns green.
* v2/azure.yaml - services + hooks (postprovision KB seed; postdeploy sample-data upload).
* v2/infra/main.bicep - resourceGroup-scoped infra (targetScope line 25); model deployments; Container Apps.
* v2/infra/main.parameters.json - model names/versions/skus/capacities.
* v2/scripts/upload_sample_data.py - postdeploy grounding seed (default benefits PDFs from repo-root data/).

### References

* .copilot-tracking/research/2026-07-01/v2-first-azd-up-deploy-research.md - primary synthesis.
* .copilot-tracking/research/subagents/2026-07-01/v2-wi07-ts6133-fix-research.md - WI-07 exact fix.
* .copilot-tracking/research/subagents/2026-07-01/v2-deploy-path-research.md - azd flow, quota, health, gates.
* .copilot-tracking/plans/logs/2026-07-01/v2-containerize-services-and-model-cleanup-log.md - WI-01…WI-09, prior deploy blockers.

### Standards References

* .github/copilot-instructions.md - Hard Rule #1 (one unit/turn), #2 (test-first), #18 (no env IDs), #19 (durable worklog tracking).
* .github/instructions/v2-frontend.instructions.md - frontend conventions for the WI-07 edit.

## Implementation Checklist

### [x] Implementation Phase 1: Unblock the frontend production build (WI-07)

<!-- parallelizable: false -->

* [x] Step 1.1: Uncomment the "Updated by" audit render in Configuration.tsx (clears `TS6133`)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 24-43)
* [x] Step 1.2: Verify the coupled vitest audit-footer test passes
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 44-60)
* [x] Step 1.3: Validate frontend build + full vitest (phase gate)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 61-75)

### [ ] Implementation Phase 2: Pre-deploy verification (read-only gates)

<!-- parallelizable: false -->

* [ ] Step 2.1: Verify model quota in the target region (WI-01)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 77-96)
* [ ] Step 2.2: Confirm azd version + login (WI-02)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 97-114)
* [ ] Step 2.3: Local docker build of all three images
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 115-132)
* [ ] Step 2.4: what-if / preview against the target resource group
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 133-156)

### [ ] Implementation Phase 3: Provision + deploy (`azd up`)

<!-- parallelizable: false -->

* [ ] Step 3.1: Create/select the azd environment + set parameters (cosmosdb, region)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 158-174)
* [ ] Step 3.2: Run `azd up` (provision + build/push/deploy all three + hooks)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 175-188)
* [ ] Step 3.3: Confirm hooks seeded index + sample data
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 189-210)

### [ ] Implementation Phase 4: Post-deploy validation (final validation)

<!-- parallelizable: false -->

* [ ] Step 4.1: Confirm all three Container Apps pulled the new image
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 212-227)
* [ ] Step 4.2: Backend health checks (`/api/health`, `/api/health/ready`)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 228-242)
* [ ] Step 4.3: Chat grounding smoke test (benefits question → citation)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 243-257)
* [ ] Step 4.4: Report outcome + next steps (worklog per Hard Rule #19)
  * Details: .copilot-tracking/details/2026-07-01/v2-first-azd-up-deploy-details.md (Lines 258-272)

## Planning Log

See .copilot-tracking/plans/logs/2026-07-01/v2-first-azd-up-deploy-log.md for discrepancy tracking, implementation paths considered, and suggested follow-on work.

## Dependencies

* azd `>= 1.18.0 != 1.23.9` (observed `1.27.0`) + `az` CLI, authenticated on the new tenant.
* Deployer holds Owner / User-Access-Administrator on the target subscription.
* Docker daemon (Phase 2.3 local builds).
* `gpt-5.1` (GlobalStandard, cap 150) + `text-embedding-3-large` (Standard, cap 100) quota in `<REGION>` (default `eastus2`).

## Success Criteria

* Frontend production image builds with no `TS6133`; full vitest green (595 passed) — Traces to: WI-07 / research §1.
* `azd up` completes SUCCESS; all three Container Apps serve freshly-built ACR images — Traces to: Objective (first azd up), research §3.
* `/api/health/ready` returns 200 in cosmos mode — Traces to: research §5.
* A default-benefits question grounds with a citation over the seeded documents — Traces to: resolved WI-03 / research §2.
