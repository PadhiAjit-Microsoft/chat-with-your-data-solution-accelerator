<!-- markdownlint-disable-file -->
# Release Changes: BUG-0094 — reasoning-panel citation markers as superscripts

**Related Plan**: bug-0094-reasoning-citation-superscripts-plan.instructions.md
**Implementation Date**: 2026-07-02

## Summary

Render inline document-citation markers in the assistant reasoning ("Thought process") panel as the
same compact superscript numbers used in the final answer body. A new pure
`superscriptReasoningCitations` helper normalizes the reasoning marker family (`[docN]`, `doc[N]`,
`docs[N]`, bare `[N]`) into `remark-supersub` `^N^` tokens; `MessageList` composes it over
`formatReasoning` and enables `enableSupersub` on the reasoning `MarkdownContent`.

## Changes

### Added

* v2/src/frontend/src/pages/chat/components/citationTokens.tsx - New shared leaf module exporting the pure `collapseConsecutiveSuperscripts(text)` helper (owns the `/\^(\d+)\^(?:\s*\^\1\^)+/g` collapse regex) consumed by both `parseAnswer` and `superscriptReasoningCitations` (resolves IV-002).
* v2/tests/frontend/pages/chat/components/citationTokens.test.tsx - 6 unit tests for the shared collapse helper (whitespace/adjacent/triple runs, distinct-adjacent no-collapse, marker-free passthrough).

### Modified

* v2/src/frontend/src/pages/chat/components/reasoningText.tsx - Added the pure exported helper `superscriptReasoningCitations(text)` plus two `UPPER_SNAKE_CASE` regex constants (`REASONING_CITATION_MARKER`, `CONSECUTIVE_DUPLICATE_SUP`); rewrites `[docN]`/`doc[N]`/`docs[N]`/bare `[N]` (1-3 digits) into ` ^N^ ` supersub tokens verbatim and collapses consecutive duplicates. `formatReasoning` untouched.
* v2/tests/frontend/pages/chat/components/reasoningText.test.tsx - Added a `describe("superscriptReasoningCitations")` block (9 cases: all four marker shapes, mixed `docs[3] and [9]`, duplicate collapse, no-marker passthrough, and the `[2026]`/`[note]` non-rewrite DD-02 guards); existing `formatReasoning` block intact. 16 tests pass.
* v2/src/frontend/src/pages/chat/components/MessageList.tsx - Imported `superscriptReasoningCitations`; the reasoning `<details>` body's model branch now renders `superscriptReasoningCitations(formatReasoning(m.reasoning))` and the `MarkdownContent` sets `enableSupersub`. Placeholder branch, `data-testid`/`data-role`, and `<details>`/`<summary>` structure untouched.
* v2/src/frontend/src/pages/chat/components/MarkdownContent.tsx - Docstring only: the reasoning-panel note now states both the answer body and the reasoning panel enable supersub (answer renders `parseAnswer`'s `^K^`, reasoning renders `superscriptReasoningCitations`'s `^N^`), with the accepted cosmetic note that a stray `^..^` in reasoning renders as `<sup>`.
* v2/tests/frontend/pages/chat/components/MessageList.test.tsx - Added one `it(...)` seeding a finished message with mixed markers (`"I checked doc[6] and docs[3] and [9]."`); asserts the reasoning panel emits `<sup>` nodes containing `6`/`3`/`9` and no longer shows the literal `doc[6]`/`docs[3]`/`[9]` text. 34 tests pass in-file.
* v2/src/frontend/src/pages/chat/components/parseAnswer.tsx - IV-002 refactor: removed the local `CONSECUTIVE_DUPLICATE_SUP_PATTERN`; now imports + calls `collapseConsecutiveSuperscripts` from `citationTokens`.
* v2/src/frontend/src/pages/chat/components/reasoningText.tsx - IV-002 refactor (further modified): removed the local `CONSECUTIVE_DUPLICATE_SUP`; `superscriptReasoningCitations` delegates the collapse to the shared `collapseConsecutiveSuperscripts`.
* v2/docs/bugs.md - BUG-0094 row (line 153) flipped `open` → `fixed` with resolved date `2026-07-02`; the "Fix direction" note replaced with a "Fix" describing the actual `superscriptReasoningCitations` helper + `enableSupersub` wiring; tail ends `**Status: fixed.**`.
* v2/docs/worklog/2026-07-02.md - BUG-0094 bug entry flipped to **fixed** with the helper + wiring summary and the green validation result.

### Removed

* None.

## Additional or Deviating Changes

* DD-01 — the bug's original "Fix direction" suggested *extending `parseAnswer`* to the reasoning body; the implementation instead added a **separate** pure `superscriptReasoningCitations` helper in `reasoningText.tsx`.
  * Reason: `parseAnswer` matches only the canonical `[docN]` shape and renumbers against `citations` for the clickable answer subset — neither fits the reasoning feed's looser markers (`doc[N]`, `docs[N]`, bare `[N]`) nor its visual-only, non-clickable need. A dedicated helper keeps `parseAnswer`'s clickable-citation contract (BUG-0016) untouched; the `enableSupersub` render path is still shared.
* DD-02 — bare `[N]` is matched with a `\d{1,3}` digit cap so a literal 4-digit bracket (e.g. `[2026]`) is not turned into a superscript. A false positive here is cosmetic (a stray `<sup>`), never a broken link. Guarded by explicit `[2026]`/`[note]` non-rewrite tests.
* The `reasoningPlaceholder` branch of the reasoning body was deliberately left un-normalized (app-generated string, no citation markers).
* Pre-existing `act(...)` warnings in `MessageInput.test.tsx` are unrelated to this change and were left as-is.
* Review remediation (2026-07-02 follow-up): `MessageList.tsx` module docstring extended to mention the `superscriptReasoningCitations` normalization (RPI-P2 Info-1); planning-log DD-02 note reconciled. IV-002 (the collapse regex `/\^(\d+)\^(?:\s*\^\1\^)+/g` duplicated in `parseAnswer.tsx` and `reasoningText.tsx`) was **resolved** — the user chose extraction over deferral, so the collapse step was factored into the shared `citationTokens.collapseConsecutiveSuperscripts` helper and both consumers refactored (behavior unchanged; 46 files / 613 tests green). WI-02 closed.

## Release Summary

BUG-0094 is fixed and closed. Seven repo files changed (all Modified; none added/removed at the source level):

* Production (3): `reasoningText.tsx` (+`superscriptReasoningCitations` helper +2 regex consts), `MessageList.tsx` (import + reasoning body composes the helper and sets `enableSupersub`), `MarkdownContent.tsx` (docstring only).
* Tests (2): `reasoningText.test.tsx` (+9 helper cases), `MessageList.test.tsx` (+1 reasoning `<sup>` render test).
* Docs (2): `bugs.md` (BUG-0094 → fixed), `worklog/2026-07-02.md` (fixed entry).

Validation: `npx tsc -b` exit 0; full frontend vitest suite **45 files / 607 tests passed**. No dependency or infrastructure changes. The visual fix ships on the next `azd deploy frontend` (live re-verify deferred as WI-01 — requires a deploy the user controls). Nothing was committed (git-ownership).
