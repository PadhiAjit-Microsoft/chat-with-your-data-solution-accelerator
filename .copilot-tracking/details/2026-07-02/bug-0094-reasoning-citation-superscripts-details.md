<!-- markdownlint-disable-file -->
# Implementation Details: BUG-0094 — reasoning-panel citation markers as superscripts

## Context Reference

Sources: .copilot-tracking/research/2026-07-02/bug-0094-reasoning-citation-superscripts-research.md
(full findings, marker-shape table, accepted tradeoff). Precedent: BUG-0016 (answer body) in
v2/docs/bugs.md line 74.

## Implementation Phase 1: Pure reasoning-citation superscript helper

<!-- parallelizable: false -->

### Step 1.1: Add `superscriptReasoningCitations(text: string): string` to reasoningText.tsx

Add a new exported pure function as a sibling of `formatReasoning` in
v2/src/frontend/src/pages/chat/components/reasoningText.tsx. It rewrites the reasoning citation
marker family into `remark-supersub` `^N^` tokens, verbatim number (no renumbering against
`citations`).

Behavior:
* Match the marker family with a single module-level `UPPER_SNAKE_CASE` regex, e.g.
  `REASONING_CITATION_MARKER = /(?:docs?\s*)?\[(?:doc)?(\d{1,3})\]/gi`. This covers `[docN]`,
  `doc[N]`, `docs[N]`, and bare `[N]`; the `\d{1,3}` cap excludes 4-digit years (DD-01).
* Replace each match with ` ^$1^ ` (leading/trailing space, mirroring parseAnswer so adjacent prose
  keeps a word boundary around the `<sup>`).
* Collapse consecutive duplicate superscripts with the same pattern parseAnswer uses:
  `/\^(\d+)\^(?:\s*\^\1\^)+/g` → `"^$1^"` (define as a sibling module constant, e.g.
  `CONSECUTIVE_DUPLICATE_SUP = ...`). Reasoning naturally repeats the same source, so this avoids
  `^3^ ^3^` stutter.
* Return the rewritten string. Input with no markers returns unchanged (regex simply no-ops).
* Do not `.trim()` here — whitespace normalization is `formatReasoning`'s job; this helper composes
  after it.

Docstring: follow the file's existing style (Pillar/Phase header already present at file top — a new
function does not add a second header). Describe *what the function is* (a pure marker→superscript
normalizer for the reasoning feed), not process narrative (Hard Rule #16). Note it emits the same
`^N^` token the answer body's `parseAnswer` produces, so both feed `remark-supersub`.

Files:
* v2/src/frontend/src/pages/chat/components/reasoningText.tsx - Add the helper + its two module
  constants; keep `formatReasoning` untouched.

Discrepancy references:
* Implements DD-02 (bare `[N]` matched with a `\d{1,3}` digit cap).

Success criteria:
* `superscriptReasoningCitations` is exported, pure, and has no React/DOM dependency.
* All imports remain at module top (Hard Rule #17); no `TYPE_CHECKING` / `__future__` (n/a in TS,
  but no lazy imports either).
* Steps 1.1 + 1.2 form one test-first unit (Hard Rule #2): the helper lands with at least a minimal
  `superscriptReasoningCitations` test stub in the same turn.

Context references:
* v2/src/frontend/src/pages/chat/components/parseAnswer.tsx (Lines 24-26) - `DOC_MARKER_PATTERN`
  and `CONSECUTIVE_DUPLICATE_SUP_PATTERN` to mirror.
* v2/src/frontend/src/pages/chat/components/reasoningText.tsx (Lines 25-31) - `formatReasoning`
  shape to sit beside.

Dependencies:
* None (leaf helper).

### Step 1.2: Add helper unit tests

Extend v2/tests/frontend/pages/chat/components/reasoningText.test.tsx with a new
`describe("superscriptReasoningCitations", ...)` block. Import the new symbol from
`@/pages/chat/components/reasoningText`.

Cases (assert on the returned string, `^N^` tokens, trimmed of the incidental spaces where helpful):
* `[doc6]` → contains ` ^6^ `.
* `doc[6]` → contains ` ^6^ ` (word `doc` consumed, not left as literal).
* `docs[3]` → contains ` ^3^ `.
* bare `[9]` → contains ` ^9^ `.
* `docs[3] and [9]` → contains both `^3^` and `^9^`, order preserved, `and` retained.
* consecutive duplicate `[doc1][doc1]` (or `doc[1] doc[1]`) → collapses to a single `^1^`.
* no markers (`"just plain reasoning."`) → returned unchanged.
* a 4-digit bracket like `[2026]` → NOT rewritten (stays literal), guarding DD-01.
* a non-numeric bracket like `[note]` → NOT rewritten.

Files:
* v2/tests/frontend/pages/chat/components/reasoningText.test.tsx - New describe block; leave the
  existing `formatReasoning` block intact.

Success criteria:
* Every marker shape from the research table has a passing assertion.
* The DD-01 guard cases (`[2026]`, `[note]`) assert non-rewrite.

Context references:
* v2/tests/frontend/pages/chat/components/reasoningText.test.tsx (Lines 10-64) - existing test
  structure and import path to follow.

Dependencies:
* Step 1.1 completion.

### Step 1.3: Run the reasoningText test file and typecheck

Run only the touched test file and the frontend typecheck to keep the loop fast.

Validation commands:
* From v2/tests/frontend: `npx vitest run pages/chat/components/reasoningText.test.tsx` - helper unit
  tests pass.
* From v2/src/frontend: `npx tsc -b` - no type errors.

Dependencies:
* Steps 1.1 + 1.2.

## Implementation Phase 2: Wire the superscript treatment into the reasoning panel

<!-- parallelizable: false -->

### Step 2.1: Compose the helper + set `enableSupersub` on the reasoning `MarkdownContent`

In v2/src/frontend/src/pages/chat/components/MessageList.tsx, update the reasoning `<details>` body
render so the model-reasoning branch flows through
`superscriptReasoningCitations(formatReasoning(m.reasoning))` and the `MarkdownContent` sets
`enableSupersub`. Add the import of `superscriptReasoningCitations` next to the existing
`import { formatReasoning } from "./reasoningText";`.

Target render (current form reads
`content={m.reasoning && m.reasoning.length > 0 ? formatReasoning(m.reasoning) : (m.reasoningPlaceholder ?? "")}`
with no `enableSupersub`):
* Wrap only the model-reasoning branch:
  `superscriptReasoningCitations(formatReasoning(m.reasoning))`. Leave the
  `m.reasoningPlaceholder ?? ""` branch untouched — the placeholder is an app-generated string with
  no citation markers, so normalizing it is a pointless no-op and keeps the transform scoped to
  model text.
* Add `enableSupersub` to that `<MarkdownContent>` element (same prop the answer body already uses).

Do not change any `data-testid` / `data-role` attributes or the `<details>`/`<summary>` structure.

Files:
* v2/src/frontend/src/pages/chat/components/MessageList.tsx - reasoning body render only (the
  `<MarkdownContent className={styles.reasoningBody} ...>` inside the `<details>` panel), plus the
  new import.

Success criteria:
* Reasoning body renders through supersub; answer body render is unchanged.
* Import is at module top (Hard Rule #17).

Context references:
* v2/src/frontend/src/pages/chat/components/MessageList.tsx (Lines 188-197) - reasoning body
  `<MarkdownContent>` to modify.
* v2/src/frontend/src/pages/chat/components/MessageList.tsx (Lines 199-203) - answer body
  `enableSupersub` render to mirror.
* v2/src/frontend/src/pages/chat/components/MessageList.tsx (Lines 69-70) - existing `formatReasoning`
  import site.

Dependencies:
* Phase 1 completion (imports the new helper).

### Step 2.2: Update the MarkdownContent docstring note

The MarkdownContent module docstring currently states the reasoning panel "leaves it off so stray
`^`/`~` in chain-of-thought stays literal." Update that sentence to describe the current behavior:
both the answer body and the reasoning panel enable supersub — the answer body to render
`parseAnswer`'s `^K^` tokens, the reasoning panel to render the `superscriptReasoningCitations`
`^N^` tokens — and note the (accepted, cosmetic) consequence that a stray `^..^` pair in reasoning
renders as a `<sup>`.

Keep it descriptive of current behavior only (Hard Rule #16 — no "changed in this turn" narrative).

Files:
* v2/src/frontend/src/pages/chat/components/MarkdownContent.tsx - docstring text only; no code
  change.

Success criteria:
* Docstring no longer claims the reasoning panel leaves supersub off.

Context references:
* v2/src/frontend/src/pages/chat/components/MarkdownContent.tsx (Lines 18-22) - the stale note.

Dependencies:
* Step 2.1 (behavior it documents).

### Step 2.3: Extend MessageList tests to assert reasoning `<sup>` render

Add a test to v2/tests/frontend/pages/chat/components/MessageList.test.tsx that seeds a finished
assistant message whose `reasoning` contains mixed markers (e.g.
`["I checked doc[6] and docs[3] and [9]."]`) and asserts the reasoning `<details>` panel:
* contains `<sup>` element(s) with the expected number text (`6`, `3`, `9`), and
* no longer contains the literal `doc[6]` / `docs[3]` / `[9]` bracketed text.

Reuse the existing `Seed` + `ChatProvider` + `act(dispatch({type:"add", message}))` harness. Query
via `screen.getByTestId("message-<id>-reasoning")` then `details.querySelectorAll("sup")` and
`details.textContent`.

Files:
* v2/tests/frontend/pages/chat/components/MessageList.test.tsx - one new `it(...)`; do not disturb
  the existing reasoning tests (they use marker-free reasoning text so supersub is a no-op for
  them).

Success criteria:
* New test asserts `<sup>` presence and literal-marker absence in the reasoning panel.
* Existing reasoning tests (concatenation, section-title drop, streaming) still pass unchanged.

Context references:
* v2/tests/frontend/pages/chat/components/MessageList.test.tsx (Lines 189-266) - reasoning-panel
  test patterns to follow.
* v2/tests/frontend/pages/chat/components/MessageList.test.tsx (Lines 38-54) - `mWithReasoning`
  fixture shape.

Dependencies:
* Steps 2.1 + 2.2.

## Implementation Phase 3: Validation and close-out

<!-- parallelizable: false -->

### Step 3.1: Run full frontend validation

Execute the full frontend validation to confirm nothing else regressed and the shared AST gates
still pass.

Validation commands:
* From v2/src/frontend: `npx tsc -b` - clean typecheck.
* From v2: `npm test` - full frontend vitest suite green (expect the two new specs to add to the
  passing count; all prior suites unchanged).
* From v2 (python venv): run the frontend-adjacent shared gates only if a `src/**` change could trip
  them — this change is TS-only, so the Python `v2/tests/shared/**` gates are not in scope. Note the
  pre-existing, unrelated `test_no_silent_excepts[src/functions/core/search_resolution.py]` failure
  is a Phase 6 concern and NOT a blocker for this frontend fix.

Dependencies:
* Phases 1 + 2 complete.

### Step 3.2: Mark BUG-0094 fixed and append the worklog entry

Per Hard Rule #19, update the two durable trackers in the same turn as the fix lands.

* v2/docs/bugs.md line 153 — flip the BUG-0094 row: `open` → `fixed`, add the fixed date
  `2026-07-02` in the resolved-date column, change "Fix direction" → "Fix", and append a
  `**Fixed 2026-07-02:**` sentence describing the `superscriptReasoningCitations` helper +
  `enableSupersub` wiring, ending `**Status: fixed.**`. Anchor the edit on the row's unique tail
  (`See [worklog/2026-07-02.md](worklog/2026-07-02.md). **Status: open.** |`) per the long-row table
  editing convention.
* v2/docs/worklog/2026-07-02.md — append a BUG-0094 fixed entry noting the pure helper, the reasoning
  `enableSupersub` wiring, and the green tsc + vitest result.

Files:
* v2/docs/bugs.md - BUG-0094 row → fixed.
* v2/docs/worklog/2026-07-02.md - fixed entry.

Success criteria:
* `grep -E "^\| BUG-0094 \|.*\| fixed \|"` matches; row is well-formed (single line, no merged
  columns).

Dependencies:
* Step 3.1 green.

### Step 3.3: Report and defer follow-on

* Report the gate results and the working-tree summary (`git status --short`); do NOT commit
  (git-ownership rule).
* Defer the **live re-verify** (WI-01 in the log): the visual fix ships on the next
  `azd deploy frontend`; confirming it on the deployed app requires a deploy the user controls.
* If validation surfaces anything beyond a minor fix, stop and report rather than expanding scope.

Dependencies:
* Steps 3.1 + 3.2.

## Dependencies

* v2/src/frontend Node toolchain (react-markdown + remark-supersub already present).
* v2/tests/frontend vitest + jsdom harness.

## Success Criteria

* Reasoning panel shows superscript citation numbers, no literal brackets; helper is pure + tested;
  tsc + full vitest green; BUG-0094 recorded fixed in bugs.md + worklog.
