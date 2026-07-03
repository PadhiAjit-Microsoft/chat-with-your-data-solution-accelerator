<!-- markdownlint-disable-file -->
# Task Review: Session doc-tracking updates + env-ID audit (2026-07-02)

## Review Metadata

| Field | Value |
|---|---|
| **Review date** | 2026-07-02 |
| **Reviewer** | Task Reviewer |
| **Reviewed work** | This session's implementation: (a) BUG-0054 close-out + cloud validation; (b) the two live-deploy review follow-up closures (embedding-model confirmation + smoke-test conversation deletion) |
| **Related review** | .copilot-tracking/reviews/2026-07-02/v2-first-azd-up-deploy-live-review.md |
| **Related changes log** | .copilot-tracking/changes/2026-07-01/v2-first-azd-up-deploy-changes.md |
| **Defect registry** | v2/docs/bugs.md → BUG-0054 |
| **Worklog** | v2/docs/worklog/2026-07-02.md |

## Scope

The session produced **no code changes** — only tracked-doc updates (bugs.md, worklog, the live-deploy changes log + review doc) plus live-cloud actions (Event Grid / queue verification, a create+delete ingestion test, a Cosmos conversation delete). RPI code-plan validation therefore does not apply; the review focuses on (1) truthfulness/consistency of the doc claims against the verified live state, and (2) a Hard Rule #18 (`no-env-specific-content-in-tracked-files`, ADR-0019) audit of every tracked file the session touched.

## Severity Summary

| Severity | Count | Notes |
|----------|-------|-------|
| Critical | 1 (RESOLVED this turn) | CR-01 — env-specific identifiers written into 4 tracked files this session |
| Major | 1 (RESOLVED 2026-07-02) | MAJ-01 — pre-existing env-ID leaks in older tracked artifacts; scrubbed across 12 files in a follow-up `/task-implement` |
| Minor | 0 | — |
| Info | 3 | doc claims verified live; registry row well-formed; Hard Rule #16 N/A (process-tracking docs are exempt) |

## Findings

### CR-01 — Env-specific identifiers leaked into tracked files (Critical) — RESOLVED

The session wrote the azd env name (`<AZD_ENV_NAME>`) and the resource suffix (`<SUFFIX>`) as **literal values** into four tracked files, violating Hard Rule #18 / ADR-0019 and the standing `azure-env-ids-never-commit` policy:

* `v2/docs/bugs.md` — BUG-0054 registry row + detail Status line + Resolution note (env name ×3, suffix ×2).
* `v2/docs/worklog/2026-07-02.md` — BUG-0054 done bullet (env name ×1).
* `.copilot-tracking/changes/2026-07-01/v2-first-azd-up-deploy-changes.md` — a deleted-conversation GUID.
* `.copilot-tracking/reviews/2026-07-02/v2-first-azd-up-deploy-live-review.md` — a deleted-conversation GUID (plus a pre-existing ACR resource name on the Phase-3 line).

**Resolution (this turn):** replaced every literal with the placeholder tokens (`<AZD_ENV_NAME>`, `<SUFFIX>`, `cr<SUFFIX>`) and dropped the deleted-conversation GUIDs (the conversation title already identifies the record; the id added no value). Post-fix grep over the four files returns **zero** env-specific tokens. No automated gate scans docs for env IDs, so this class relies on review discipline — flagged here so the pattern is not repeated.

### MAJ-01 — Pre-existing env-ID leaks in older tracked artifacts (Major) — RESOLVED (2026-07-02)

A repo-wide grep surfaced tracked, **pre-existing** (not this session) env-specific identifiers referencing a now-retired deployment. **Correction to the first-pass characterization:** the GUID `c6a89b2d-…` originally flagged here as "an Entra object id" is in fact the **built-in "Storage Queue Data Message Sender" role-definition GUID** (it appears in `infra/modules/.../storage-account.bicep` and is described as a role id in every research file) — a tenant-independent, public value and an explicit ADR-0019 carve-out, so it was correctly left in place. Likewise `${solutionSuffix}` / `<SUFFIX>` template forms and Azure region names (`eastus2`/`uksouth`) are not leaks and were left.

**Resolved 2026-07-02 (FU-01):** scrubbed the real literal leaks across **12 tracked files** — the retired suffix → `<SUFFIX>`, the older v1-style suffix → `<OLD_SUFFIX>` (preserving the deliberate old-vs-current contrast in the Step 6.6 cleanup narrative), the env name → `<AZD_ENV_NAME>`, the resource group → `<RESOURCE_GROUP>`, embedded FQDNs → placeholder-suffixed forms, plus two adjacent classes discovered during the scrub: the deployer **UPN** (`<AZURE_PRINCIPAL_UPN>`) and leaked **subscription names** (`<AZURE_SUBSCRIPTION_NAME>`, both the retired and the current subscription). Files scrubbed: the 2026-06-29 changes + details + plan, four research files (2026-06-28/29/30), and two `v2/docs` product docs (`bugs.md`, `worklog/2026-06-16.md`) that carried the legacy index name. Post-scrub repo-wide grep for every token class (suffixes, env name, RG, subscription names, UPN, sub/tenant/object ids) returns **zero** matches.

## Validation

| Check | Result |
|---|---|
| BUG-0054 live claims (Event Grid → `blob-events`, create 263→264, delete 264→263, queues at 0) | ✅ Verified live during implementation |
| Embedding-model claim (`text-embedding-3-small`, 1536-dim, azd-env-pinned) | ✅ Verified live (`az cognitiveservices ... deployment list`) |
| Smoke-test conversation deleted | ✅ Verified (re-list showed it gone; 1 unrelated test conversation remains, flagged) |
| bugs.md BUG-0054 registry row | ✅ Well-formed single line, `fixed`, resolved 2026-07-02; open-bug count = 7 |
| Placeholder audit of the 4 session files | ✅ Zero env-specific tokens post-fix |
| Hard Rule #16 (process narrative) | N/A — applies to `v2/src/**` `.py`; process-tracking docs are exempt |
| git-ownership | ✅ Honored — nothing committed |

## Follow-Up Work

* **FU-01 (Major)** — ✅ **DONE (2026-07-02).** Env-ID scrub completed across 12 tracked files (see MAJ-01 Resolution). Repo-wide grep is clean of every env-specific token class.
* **FU-02 (Low)** — ✅ **DONE (2026-07-02).** Added `v2/tests/shared/test_no_env_specific_content.py` — a denylist-from-local-env gate. It reads the developer's gitignored azd `.env` (never hard-coding a secret), builds a denylist of the real env-specific values it holds (subscription / tenant / RG / env-name / resource-name vars + a **derived** resource suffix — 10 entries against the current env), and asserts no git-tracked file contains any of them. It runs on the dev machine where leaks originate and skips gracefully when no local env is present (CI); the parsing / suffix-derivation / detection helpers are unit-tested so the mechanism stays covered regardless. Full shared-gate suite green (1039 passed). ADR-0019 is no longer discipline-only for the current environment.
* **Carried from the live-deploy review** — delete the second guest test conversation (`tell me about employee benefits`) if unwanted; remove the three temporary self-granted data-plane roles (`Storage Queue Data Contributor`, `Search Index Data Reader`, `Cosmos DB Built-in Data Contributor`) used for verification.

## Scrub Re-review (2026-07-02)

Independent verification of the FU-01 / MAJ-01 env-ID scrub (12 tracked files):

* **Tracked tree clean — VERIFIED.** A gitignore-respecting repo-wide grep for every token class (retired + v1 suffixes, env name, RG, subscription names, deployer UPN, sub/tenant/object ids) returns **zero** matches.
* **No corruption from the many edits — VERIFIED.** Spot-checked the two high-edit files: the changes-log "Live outputs" line reads cleanly with all placeholders substituted (`<SUFFIX>`/`<RESOURCE_GROUP>`/placeholder FQDNs); the details Step 6.6 block is intact.
* **Old-vs-current distinction preserved — VERIFIED.** details Step 6.6 still says "Delete ONLY the `<OLD_SUFFIX>`-suffixed resources … NEVER touch any `<SUFFIX>`-suffixed (current v2) resource," and the validation queries key `<OLD_SUFFIX>` (delete) vs `<SUFFIX>` (keep) correctly — the safety-critical contrast was not flattened.
* **Gitignored hits correctly excluded — VERIFIED.** A non-gitignore filesystem sweep surfaced further real values in `.azure/` scratch, `.copilot-tracking/plans/logs/`, and a v1 `code/.../local.settings.json`; `git ls-files`/`check-ignore` confirmed all are **gitignored** (untracked) — out of Hard Rule #18 scope, correctly left.
* **No new leaks in the scrub's own tracking text — VERIFIED.** The review-log / changes-log / worklog additions are placeholder-only (covered by the clean grep).
* **RV-01 (this re-review) — FIXED.** The Severity Summary's Major row was stale (`OPEN, follow-up`) after MAJ-01 was resolved; corrected to `RESOLVED 2026-07-02`.
* **Judgment calls — ACCEPTED.** (a) Genericizing subscription *names* (not on Hard Rule #18's explicit ID list) is a defensible tightening under ADR-0019's "only generic development and deployment guidance" spirit — names are org-identifying. (b) The non-standard placeholders `<OLD_SUFFIX>` and `<AZURE_SUBSCRIPTION_NAME>` are self-documenting and necessary (the former to preserve the two-suffix contrast); acceptable extensions of the documented token set. (c) The role-def GUID `c6a89b2d…` correction (carve-out, not an Entra object id) is accurate.

## Overall Status

✅ **Complete — Critical finding resolved in-turn.** The session's doc claims are truthful and consistent with the verified live state, and the BUG-0054 registry row + worklog are well-formed. The one Critical finding (CR-01, env-ID leak I introduced) was remediated this turn; the four session-edited tracked files are now placeholder-clean. The Major follow-up (MAJ-01, pre-existing leaks in older tracking artifacts) was subsequently **resolved in a follow-up `/task-implement`** — a 12-file env-ID scrub (retired + v1 suffixes, env name, RG, subscription names, deployer UPN) with a clean repo-wide grep; the mischaracterized "Entra object id" was corrected to a built-in role-definition GUID (an ADR-0019 carve-out, left in place). Nothing committed (git-ownership).
