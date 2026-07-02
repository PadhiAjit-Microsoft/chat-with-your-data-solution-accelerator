<!-- markdownlint-disable-file -->
# Release Changes: CWYD v2 — First `azd up` Deploy Path

**Related Plan**: v2-first-azd-up-deploy-plan.instructions.md
**Implementation Date**: 2026-07-01

## Summary

Take CWYD v2 from green-but-undeployed to a first successful `azd up`: unblock the one hard frontend build blocker (WI-07 `TS6133`), run read-only pre-deploy verification, provision + deploy all three container services, then validate the live deployment grounds on seeded sample data.

## Phase Status

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | Unblock the frontend production build (WI-07) | Complete |
| 2 | Pre-deploy verification (read-only gates) | Blocked (operator / live Azure) |
| 3 | Provision + deploy (`azd up`) | Blocked (operator-gated live deploy → Task Reviewer) |
| 4 | Post-deploy validation (final) | Blocked (depends on Phase 3) |

## Changes

### Added

* (none yet)

### Modified

* v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx - Phase 1 (WI-07): uncommented the 6-line "Updated by" audit render (deleted only the `{/* ` and ` */}` JSX comment markers at lines 1040/1045) so `formatActor` (line 628) gains a live call site. Clears `TS6133` under `noUnusedLocals` and turns the pre-existing failing vitest green. `RuntimeConfig.updated_by` (admin.tsx:278) already exists; no model/import/type change. One-unit edit only.

### Removed

* (none yet)

## Additional or Deviating Changes

* Execution cadence (2026-07-01): user ran `/task-implement`. **Phase 1 executed autonomously + validated green.** Phases 2-4 are operational (live Azure + docker daemon + operator-gated login/deploy) and are handed to the operator + Task Reviewer per the plan's handoff note; they are NOT executed this turn.
* Operational readiness snapshot (read-only diagnostics, 2026-07-01):
  * `azd version` = **1.27.0** — satisfies the pin `>= 1.18.0 != 1.23.9` (Step 2.2 version part CONFIRMED).
  * **Docker daemon not running locally** (Docker Desktop Linux engine unavailable) — Step 2.3 (local docker builds) cannot run here. Mitigated: `v2/azure.yaml` sets `remoteBuild: true`, so ACR builds all three images during `azd up` regardless; the local build is only an optional pre-check.
  * **`az` / `azd` ARM tokens expired** (`AADSTS700082`, 12h inactivity) — `az account show` returns cached account data, but any ARM call (quota list, what-if) requires an interactive `az login` / `azd auth login`. Blocks Steps 2.1 (quota) + 2.4 (what-if) + Phase 3 until the operator re-authenticates. Not run autonomously (interactive login involves credentials).
  * **Target subscription reported state `Warned`** — a billing/compliance flag that can restrict provisioning. Should be resolved before `azd up`.* Review-finding cleanup (2026-07-01, `/task-implement` addressing the review doc): fixed RPI **Minor-2** — corrected the off-by-one `noUnusedLocals` line reference (`tsconfig.json:8` → `:9`) in `.copilot-tracking/research/subagents/2026-07-01/v2-wi07-ts6133-fix-research.md` (2 references). RPI **Minor-1** was already closed during review (full suite 595/0). No open review findings remain that are autonomously actionable; the deferred deploy follow-ups (quota, `azd provision --preview`, `azd up`, post-deploy validation) require operator Azure auth + a resolved `Warned` subscription and belong to the live-deploy turn.
## Release Summary

_Pending — written after the final phase._
