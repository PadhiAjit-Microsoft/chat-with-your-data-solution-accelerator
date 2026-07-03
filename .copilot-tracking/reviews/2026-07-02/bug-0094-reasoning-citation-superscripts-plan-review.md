<!-- markdownlint-disable-file -->
# Review Log: BUG-0094 — reasoning-panel citation markers as superscripts

## Review Metadata

| Field | Value |
|---|---|
| **Review date** | 2026-07-02 |
| **Plan** | .copilot-tracking/plans/2026-07-02/bug-0094-reasoning-citation-superscripts-plan.instructions.md |
| **Changes log** | .copilot-tracking/changes/2026-07-02/bug-0094-reasoning-citation-superscripts-changes.md |
| **Research** | .copilot-tracking/research/2026-07-02/bug-0094-reasoning-citation-superscripts-research.md |
| **Planning log** | .copilot-tracking/plans/logs/2026-07-02/bug-0094-reasoning-citation-superscripts-log.md |
| **Reviewer** | Task Reviewer |

## Summary of Findings

All three plan phases **Verified** by independent RPI Validators; implementation quality **Passed**.
Zero Critical, zero Major. One cosmetic changes-log nit was fixed during the review; the sole
optional DRY observation (IV-002) was subsequently **extracted at the user's request** (see the
IV-002 resolution below).

| Severity | Count | Notes |
|---|---|---|
| Critical | 0 | — |
| Major | 0 | — |
| Minor | 0 open (2 resolved) | Resolved: IV-002 collapse-regex duplication (extracted to `citationTokens.collapseConsecutiveSuperscripts`, both consumers refactored) + RPI-P1 MINOR-1 changes-log `### Added` `_pending_` → `None`. |
| Info | 5 | DD-02 digit cap + bare-`[N]` accepted class + header phase-tail + `lastIndex`-safe confirmation + `document[5]` prefix boundary. |

## Applicable Conventions (applyTo match)

Changed source lives under `v2/src/frontend/**` and `v2/tests/frontend/**`, so these apply:

* .github/instructions/v2-frontend.instructions.md (`v2/src/frontend/**`)
* .github/instructions/v2-tests.instructions.md (`v2/tests/**`, `v2/src/**/tests/**`)
* .github/copilot-instructions.md (always-on Hard Rules)

## RPI Validation (per plan phase)

Validation files under .copilot-tracking/reviews/rpi/2026-07-02/.

### Phase 1 — Pure reasoning-citation superscript helper

**Verified** (0C / 0M / 1 Minor / 2 Info). `superscriptReasoningCitations` is exported, pure, and a
character-for-character match to the Step 1.1 spec (`REASONING_CITATION_MARKER` with the `\d{1,3}`
cap; verbatim `^$1^`; identical collapse regex to `parseAnswer`). `formatReasoning` untouched; all 9
required test cases present (incl. `[2026]`/`[note]` guards). MINOR-1 (changes-log `### Added`
`_pending_`) fixed this turn. See
[plan-001-validation.md](.copilot-tracking/reviews/rpi/2026-07-02/bug-0094-reasoning-citation-superscripts-plan-001-validation.md).

### Phase 2 — Wire the superscript treatment into the reasoning panel

**Verified** (0C / 0M / 0 Minor / 4 Info). Only the model-reasoning branch is wrapped; placeholder
branch, `data-testid`/`data-role`, and `<details>`/`<summary>` untouched; `enableSupersub` present on
the reasoning `MarkdownContent`; answer body unchanged. `MarkdownContent` docstring is present-tense,
process-narrative-free. New MessageList `<sup>` render test present; existing reasoning tests intact.
See
[plan-002-validation.md](.copilot-tracking/reviews/rpi/2026-07-02/bug-0094-reasoning-citation-superscripts-plan-002-validation.md).

### Phase 3 — Validation and close-out

**Verified** (0C / 0M / 0 Minor / 2 Info). The validator **re-executed the gates live**: `npx tsc -b`
exit 0 and full frontend vitest **45 files / 607 tests** — matching the changes-log claim. `bugs.md`
BUG-0094 row is a well-formed single line, `fixed` with resolved date 2026-07-02, `**Status: fixed.**`,
and its Fix text reflects the DD-01 separate-helper implementation. Worklog entry present. No Hard
Rule #18 leaks; git-ownership honored. See
[plan-003-validation.md](.copilot-tracking/reviews/rpi/2026-07-02/bug-0094-reasoning-citation-superscripts-plan-003-validation.md).

## Implementation Quality Findings

**Passed** (0C / 0M / 1 Minor optional / 3 Info). Full report:
[impl-validation.md](.copilot-tracking/reviews/impl/2026-07-02/bug-0094-reasoning-citation-superscripts-impl-validation.md).

* IV-001 — module-level `/g` regex `lastIndex` reuse: **verified SAFE** (constants consumed only via
  `.replace`; zero `.test()`/`.exec()`; spec resets `lastIndex` per `.replace`). Not a defect.
* IV-002 — the collapse regex `/\^(\d+)\^(?:\s*\^\1\^)+/g` is duplicated in `parseAnswer.tsx:26`
  and `reasoningText.tsx:42` (Minor, optional). Extracting a shared leaf for one literal is
  over-engineering today; already tracked as WI-02 (unify only if a third citation surface appears).
* IV-003/IV-004 — bare-`[N]` false-positive class and the 3-digit cap are accepted DD-02 tradeoffs,
  visual-only and test-guarded.

## Validation Command Outputs

| Command | cwd | Result |
|---|---|---|
| `npx tsc -b` | v2/src/frontend | **exit 0** (reviewer + Phase-3 validator) |
| `npx vitest run` (full suite) | v2/tests/frontend | **45 files / 607 tests passed** |
| `get_errors` on 5 changed files | — | **No errors found** |

## Missing Work and Deviations

* No missing work — every plan step (1.1–3.3) is implemented and checked complete.
* Deviations are intentional and recorded: **DD-01** (separate `superscriptReasoningCitations` helper
  rather than extending `parseAnswer`, to protect the answer-body clickable-citation contract);
  **DD-02** (bare `[N]` with a `\d{1,3}` cap). Both are documented in the planning log and reflected in
  the bugs.md Fix text.

## Remediation (2026-07-02 follow-up)

Addressed the actionable review findings:

* **RPI-P2 Info-1 — ADDRESSED.** `MessageList.tsx` module docstring now states the reasoning body's
  inline citation markers are normalized to `^N^` superscripts by `superscriptReasoningCitations`
  and rendered via `enableSupersub` (it previously described only `formatReasoning`). Docstring-only,
  present-tense (Hard Rule #16 clean); `get_errors` reports no errors.
* **DD-numbering nit (RPI-P1 Info-2 / RPI-P3 Info-2) — ADDRESSED.** The planning-log DD-02 validation
  note is reconciled: the earlier `DD-01` mis-citations in the research doc + details Step 1.1 were
  already corrected to `DD-02` (confirmed by RPI Phase 1), so the note no longer says "should be
  corrected".
* **RPI-P1/P3 MINOR-1 / INFO-1 — RESOLVED (during review).** Changes-log `### Added` `_pending_` →
  `None`.
* **Info (accepted, no action) —** DD-02 digit cap + bare-`[N]` false-positive class (intended,
  test-guarded); `lastIndex` reuse (verified safe — `.replace`-only, spec resets per call);
  `document[5]` prefix boundary (correct); `test_no_process_narrative_in_src.py` `.py`-only scope
  (factual clarification, not a defect).
* **IV-002 (Minor) — RESOLVED (2026-07-02, user chose extraction).** The duplicated collapse regex
  `/\^(\d+)\^(?:\s*\^\1\^)+/g` was factored into a new shared leaf module `citationTokens.tsx`
  exporting the pure `collapseConsecutiveSuperscripts(text)` helper (test-first, 6 cases); both
  `parseAnswer` and `superscriptReasoningCitations` now delegate to it and dropped their local
  copies. The structural change was confirmed with the user first (Hard Rule #10). Behavior
  unchanged — full suite **46 files / 613 tests** green. WI-02 closed.

## Follow-Up Work

Deferred from scope (pre-existing in the plan):
* **WI-01** (medium) — live re-verify the superscripts on the deployed frontend after the next
  `azd deploy frontend`. Requires a deploy the user controls.

Discovered during review:
* **WI-02** (low) — ✅ **DONE (2026-07-02).** The shared collapse regex was extracted to
  `citationTokens.collapseConsecutiveSuperscripts` and both consumers refactored (IV-002 resolved).
  The marker-matching regexes (`DOC_MARKER_PATTERN` vs `REASONING_CITATION_MARKER`) stay
  intentionally separate by design, so no further unification is needed.

## Re-review Validation (2026-07-02)

Focused re-review of the remediation delta (the full 3-phase RPI + impl validation was already
Verified and is not re-run per the resumption guidance — the implementation substance is unchanged).

* **`MessageList.tsx` docstring (RPI-P2 Info-1 fix) — VERIFIED.** The added lines describe the
  reasoning body's `superscriptReasoningCitations` → `^N^` → `enableSupersub` flow in plain present
  tense; no unit IDs / dates / "changed" / phase narrative, so Hard Rule #16 is honored.
  Docstring-only (no behavioral change).
* **Gates re-run live — GREEN.** `npx tsc -b` exit 0; full frontend vitest **45 files / 607 tests**
  passed — confirming the comment-only edit did not regress the build or suite.
* **Tracking-doc consistency — VERIFIED.** details Step 1.1 now cites `DD-02`; the planning-log DD-02
  note is reconciled (no residual "should be corrected"); `bugs.md` BUG-0094 remains a well-formed
  single-line `fixed` row. The lone remaining "should be corrected" string is inside
  `plan-001-validation.md` — the RPI-P1 validator's historical finding record, correctly left as the
  audit trail.
* **IV-002 — OPEN by decision.** The duplicated collapse regex remains deferred as WI-02; extraction
  is a structural change (Hard Rule #10) awaiting user opt-in. No new findings surfaced.

## IV-002 Extraction Re-review (2026-07-02)

Independent verification of the IV-002 resolution delta (the shared `collapseConsecutiveSuperscripts`
extraction the user opted into):

* **New `citationTokens.tsx` — VERIFIED.** Pure `(string) => string` helper owning the single
  `/\^(\d+)\^(?:\s*\^\1\^)+/g` regex; carries the `Pillar:`/`Phase:` header (Hard Rule #3) and
  descriptive present-tense docstrings with no process narrative (Hard Rule #16). Filename is
  `camelCase.tsx` per the utility convention (ADR-0013).
* **`parseAnswer.tsx` / `reasoningText.tsx` refactor — VERIFIED.** Both dropped their local collapse
  constant and delegate to the shared helper; the marker regexes (`DOC_MARKER_PATTERN`,
  `REASONING_CITATION_MARKER`) are retained, so behavior is identical. New imports sit at module top
  (Hard Rule #17). Grep confirms **no dead references** to the removed `CONSECUTIVE_DUPLICATE_SUP*`
  constants outside `citationTokens.tsx`.
* **Test-first — VERIFIED.** `citationTokens.test.tsx` (6 cases) landed with the helper.
* **Gates re-run — GREEN.** `tsc -b` exit 0; the four affected suites (parseAnswer 13, reasoningText
  16, citationTokens 6, MessageList 34) = **69 tests** pass, confirming behavior preservation (the
  full suite was 46 files / 613 green in the implementing turn on the identical code state).
* **No new findings.**

## Overall Status

✅ **Complete (fully verified).** All plan items verified across three RPI phases and
implementation-quality review; the remediation delta re-validated green (tsc 0; 45 files / 607
tests). Zero Critical / zero Major. Review findings remediated: two doc-accuracy nits fixed
(`MessageList.tsx` docstring + planning-log DD-02 note), the cosmetic changes-log nit resolved, and
IV-002 **resolved** — the duplicated collapse regex was extracted to the shared
`citationTokens.collapseConsecutiveSuperscripts` helper at the user's request and both consumers
refactored (behavior unchanged; full suite **46 files / 613 tests** green). All review findings are
now closed. BUG-0094 is correctly closed in bugs.md + worklog. Nothing committed (git-ownership) —
ready for the user to commit and deploy.
