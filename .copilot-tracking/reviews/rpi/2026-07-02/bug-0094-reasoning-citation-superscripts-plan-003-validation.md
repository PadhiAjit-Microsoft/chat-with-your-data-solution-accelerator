<!-- markdownlint-disable-file -->
# RPI Validation: BUG-0094 — Phase 3 (Validation and close-out)

**Plan**: `.copilot-tracking/plans/2026-07-02/bug-0094-reasoning-citation-superscripts-plan.instructions.md`
**Details**: `.copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md`
**Changes log**: `.copilot-tracking/changes/2026-07-02/bug-0094-reasoning-citation-superscripts-changes.md`
**Planning log**: `.copilot-tracking/plans/logs/2026-07-02/bug-0094-reasoning-citation-superscripts-log.md`
**Phase**: 3 — Validation and close-out (Steps 3.1–3.3)
**Validated**: 2026-07-02
**Status**: **Verified** (Passed)

## Executive summary

Phase 3 is **fully implemented and accurate**. Every claim in the changes log and both
durable trackers was independently confirmed against the actual repo state, and the two
gate claims (`tsc -b` exit 0; full frontend vitest 45 files / 607 tests) were **re-executed
live** and matched exactly. The `bugs.md` BUG-0094 row is a well-formed single-line `fixed`
row whose Fix text accurately reflects the **DD-01 separate-helper** implementation (not the
original "extend `parseAnswer`" direction). No Hard Rule #18 leaks were introduced. All
Phase 3 checkboxes are marked complete and nothing was committed (git-ownership honored).

## Finding counts by severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| Major    | 0 |
| Minor    | 0 |
| Info     | 2 |

## Step-by-step verification

### Step 3.1 — Full frontend validation — VERIFIED (re-executed)

* Changes-log claim: `npx tsc -b` exit 0; full frontend vitest **45 files / 607 tests** passing.
* Re-run `npx tsc -b` from `v2/src/frontend` → `TSC_EXIT=0`.
* Re-run `npm test` from `v2` → `Test Files 45 passed (45)` / `Tests 607 passed (607)`.
  New specs present in the run: `pages/chat/components/MessageList.test.tsx (34 tests)`
  passing; `reasoningText.test.tsx` `superscriptReasoningCitations` block passing.
* Evidence: live terminal output (this session).

### Step 3.2 — BUG-0094 marked fixed + worklog entry — VERIFIED

`v2/docs/bugs.md` line 153:

* Well-formed **single line** (2,145 chars, no embedded newline / no merged columns);
  adjacent BUG-0093 (line 152) and BUG-0095 (line 154) are likewise single-line.
* Correct 7-column schema: `| BUG-0094 | 2026-07-02 | 2026-07-02 | frontend | medium | fixed | …`
  — resolved date `2026-07-02` present, status `fixed`.
* Tail ends `… See [worklog/2026-07-02.md](worklog/2026-07-02.md). **Status: fixed.** |`.
* Fix text accurately reflects **DD-01** (separate helper, not extend `parseAnswer`):
  "added a pure `superscriptReasoningCitations` helper in [reasoningText.tsx] that normalizes
  the reasoning marker family (`[docN]`, `doc[N]`, `docs[N]`, bare `[N]`; 1-3 digits, so a
  4-digit `[2026]` stays prose) into ` ^N^ ` `remark-supersub` tokens … `MessageList.tsx`
  composes it over `formatReasoning` and sets `enableSupersub` …". The DD-02 `\d{1,3}` cap is
  called out ("1-3 digits … a 4-digit `[2026]` stays prose"). The `parseAnswer` mention in the
  root-cause sentence is correct *context* (answer body precedent), not the fix direction.
* Fix reflects real code (cross-checked): `reasoningText.tsx` L38 `REASONING_CITATION_MARKER =
  /(?:docs?\s*)?\[(?:doc)?(\d{1,3})\]/gi;`, L42 `CONSECUTIVE_DUPLICATE_SUP`, L58
  `export function superscriptReasoningCitations`; `MessageList.tsx` L70 import, L193 compose,
  L198/L205 `enableSupersub`.

`v2/docs/worklog/2026-07-02.md` line 33: BUG-0094 **fixed** entry present in the Bugs section,
consistent with bugs.md and the changes log — names the pure helper, the marker family, the
`^N^` supersub tokens (verbatim number), `MessageList` composing over `formatReasoning` +
`enableSupersub`, and "tsc clean, full frontend vitest green (45 files / 607 tests; +9 helper
cases, +1 render test)".

### Step 3.3 — Defer live re-verify + no commit — VERIFIED

* Deferral documented: changes-log Release Summary — "live re-verify deferred as WI-01
  (requires a deploy the user controls)"; planning log WI-01 (`ca-frontend-<SUFFIX>` after
  `azd deploy frontend`, placeholder used).
* No commit: `git status --short` shows all **7** BUG-0094 files as ` M` (modified, unstaged,
  uncommitted); last commit `f1d53044` = "Fix history and citation links on split-host deploy"
  (BUG-0092/BUG-0095), i.e. **no** BUG-0094 commit. git-ownership honored.

## Cross-cutting checks

### Changes-log Release Summary + DD-01/DD-02 accuracy — VERIFIED

* "Seven repo files changed (all Modified)" matches `git status` exactly: 3 prod
  (`reasoningText.tsx`, `MessageList.tsx`, `MarkdownContent.tsx`), 2 test
  (`reasoningText.test.tsx`, `MessageList.test.tsx`), 2 docs (`bugs.md`,
  `worklog/2026-07-02.md`).
* DD-01 (separate helper, not extend `parseAnswer`) — confirmed in code; `parseAnswer`
  clickable-citation contract untouched.
* DD-02 (`\d{1,3}` bare-`[N]` cap) — confirmed at `reasoningText.tsx` L38, guarded by explicit
  non-rewrite tests: `[2026]` (`reasoningText.test.tsx` L107) and `[note]` (L113).
* MessageList `<sup>` render test present: `MessageList.test.tsx` L230 "renders reasoning
  citation markers as superscripts in the panel" — seeds `"I checked doc[6] and docs[3] and
  [9]."`, asserts `<sup>` text contains `6`/`3`/`9` and no literal `doc[6]`/`docs[3]`.

### Hard Rule #18 (no env-specific leaks in changed tracked files) — VERIFIED

* GUID / `subscriptions/` / `tenantId` / `AZURE_SUBSCRIPTION` / `resourceGroup` scan across
  `bugs.md`, `worklog/2026-07-02.md`, and the changes log:
  * changes log + worklog: **zero** matches.
  * `bugs.md`: all 6 matches fall **outside** the BUG-0094 row and are carve-outs — the
    placeholder-warning header (L46), the all-zeros default user
    `00000000-0000-0000-0000-000000000000` (L105/L862/L876), and the **built-in role-definition
    GUID** `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd` (Cognitive Services OpenAI User, L484/L488).
* The BUG-0094 row and worklog entry reference only file paths / helper names; WI-01 uses the
  `<SUFFIX>` placeholder. No subscription / tenant / suffix / RG literals introduced.

### Plan Phase 3 checkboxes complete — VERIFIED

Plan shows `### [x] Implementation Phase 3` with `* [x]` on Steps 3.1, 3.2, 3.3 (Phases 1 and
2 likewise `[x]`).

## Info-level findings (non-blocking, out of Phase 3 remediation scope)

* **INFO-1 — changes log "Added" subsection still reads `_pending_`.** In
  `.copilot-tracking/changes/2026-07-02/bug-0094-…-changes.md` the `### Added` list shows
  `* _pending_`, while `### Removed` correctly reads `None` and the Release Summary states all
  seven files are Modified (none added/removed). Cosmetic only — the Modified list and Release
  Summary are authoritative and accurate; the `_pending_` placeholder should have been set to
  `None` at close-out.
* **INFO-2 — DD-numbering cross-reference drift (Phase 1 artifact, already flagged).** The
  planning log's own validation note records that details Step 1.1 cites the digit-cap tradeoff
  as `DD-01` whereas it is canonically `DD-02`, and that `DR-01`/`DD-02` framing differs from
  the parent task summary. This is a pre-existing planning-doc traceability note (raised by the
  plan-validator during Phase 1), not a Phase 3 close-out defect, and does not affect the
  shipped fix or the durable trackers.

## Coverage assessment

Phase 3 coverage is **complete**. All three steps are implemented; both gate claims were
independently re-executed and matched; both durable trackers (Hard Rule #19) are updated and
mutually consistent with the changes log and the source code; git-ownership and Hard Rule #18
are honored. No Critical/Major/Minor gaps found.

## Recommended next validations (not performed here)

* [ ] **WI-01 (deferred by design)** — live re-verify that reasoning-panel superscripts render
  on the deployed frontend after the next `azd deploy frontend` (requires an operator-controlled
  deploy; blocked on BUG-0093 backend health being green).
* [ ] Optional cosmetic tidy: set the changes-log `### Added` line from `_pending_` to `None`
  (INFO-1).

## Clarifying questions

None — all Phase 3 claims were resolvable from the repo state and live gate re-runs.
