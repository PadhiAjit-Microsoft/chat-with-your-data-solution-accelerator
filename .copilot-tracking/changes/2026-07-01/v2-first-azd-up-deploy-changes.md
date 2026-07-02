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
* Live-deploy review follow-ups closed (2026-07-02, `/task-implement` addressing `.copilot-tracking/reviews/2026-07-02/v2-first-azd-up-deploy-live-review.md`): the two remaining non-Critical follow-ups were resolved (the Critical BUG-0093 was already fixed + verified). **Minor (embedding model) — CONFIRMED `text-embedding-3-small` is intended:** the deployed model is `text-embedding-3-small` v1 (native 1536-dim, matching the `cwyd-index`), deliberately pinned via `AZURE_ENV_EMBEDDING_MODEL_NAME=text-embedding-3-small` in the azd env (overriding the bicep `-large` default); grounding is verified end-to-end, so no infra change is needed — the research doc's `-large` was an assumption, not the intent. **Cleanup — smoke-test conversation DELETED:** removed the grounding smoke-test conversation (`"What does the Northwind Health Plus plan cover?"`, user `local-dev`) + its 2 messages from the Cosmos `conversations` container (via the Cosmos SDK under a temporary self-granted data role). One additional test conversation (`"tell me about employee benefits"`, guest user) remains and is flagged for the operator.
* Env-ID scrub (2026-07-02, `/task-implement` addressing FU-01 / MAJ-01 in `.copilot-tracking/reviews/2026-07-02/v2-live-deploy-followups-and-env-id-audit-review.md`): genericized pre-existing env-specific identifiers across **12 tracked files** so no tracked file carries real deployment values (Hard Rule #18 / ADR-0019). Retired suffix → `<SUFFIX>`, older v1-style suffix → `<OLD_SUFFIX>` (preserving the Step 6.6 old-vs-current cleanup contrast), env name → `<AZD_ENV_NAME>`, resource group → `<RESOURCE_GROUP>`, embedded FQDNs → placeholder-suffixed, deployer UPN → `<AZURE_PRINCIPAL_UPN>`, subscription names → `<AZURE_SUBSCRIPTION_NAME>`. Files: the 2026-06-29 changes/details/plan, four research files (2026-06-28/29/30), the 2026-07-02 live-deploy review, and two `v2/docs` product docs (`bugs.md`, `worklog/2026-06-16.md`). Built-in role-definition GUIDs (`c6a89b2d…` etc.) and Azure region names were correctly left (ADR-0019 carve-outs) — the review's earlier "Entra object id" note was a mischaracterization, now corrected. Post-scrub repo-wide grep for every token class returns zero matches.
* Automated env-ID gate added (2026-07-02, addressing review finding FU-02): new `v2/tests/shared/test_no_env_specific_content.py` enforces Hard Rule #18 / ADR-0019. It reads the developer's gitignored `v2/.azure/<env>/.env`, builds a denylist of the real env-specific values (subscription / tenant / RG / env name / resource-name vars + a derived resource suffix — 10 entries on the current env) without hard-coding any secret, and asserts no git-tracked file (`git ls-files`) contains any of them. Runs on the dev machine (leak origin), skips when no local env is present (fresh clone / CI); the dotenv-parse, suffix-derivation, denylist-build, and leak-detection helpers are unit-tested so the mechanism stays covered when the live check skips. 5/5 module tests pass; full `tests/shared` suite green (1039 passed, 1 pre-existing skip).
## Release Summary

_Pending — written after the final phase._
