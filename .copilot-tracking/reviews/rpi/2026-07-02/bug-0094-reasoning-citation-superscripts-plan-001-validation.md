<!-- markdownlint-disable-file -->
# RPI Validation — BUG-0094 Phase 1 (Pure reasoning-citation superscript helper)

- **Plan**: `.copilot-tracking/plans/2026-07-02/bug-0094-reasoning-citation-superscripts-plan.instructions.md`
- **Details**: `.copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md`
- **Research**: `.copilot-tracking/research/2026-07-02/bug-0094-reasoning-citation-superscripts-research.md`
- **Changes log**: `.copilot-tracking/changes/2026-07-02/bug-0094-reasoning-citation-superscripts-changes.md`
- **Planning log**: `.copilot-tracking/plans/logs/2026-07-02/bug-0094-reasoning-citation-superscripts-log.md`
- **Phase validated**: 1 (Steps 1.1–1.3)
- **Validation date**: 2026-07-02
- **Status**: **Verified**

## Scope

Phase 1 through-line only: the pure exported `superscriptReasoningCitations(text: string): string`
helper in `reasoningText.tsx` (sibling of an untouched `formatReasoning`) and its
`describe("superscriptReasoningCitations")` vitest block in `reasoningText.test.tsx`. Phases 2–3
(MessageList wiring, docstring, close-out) are out of this validation's scope.

## Finding Counts

| Severity | Count |
|----------|-------|
| Critical | 0 |
| Major    | 0 |
| Minor    | 1 |
| Info     | 2 |

## Plan-Item → Change Coverage

| Step | Plan requirement | Status | Evidence |
|------|------------------|--------|----------|
| 1.1 | Add exported pure `superscriptReasoningCitations(text)` sibling to `formatReasoning` | Verified | reasoningText.tsx:61-66 (`export function`) |
| 1.1 | Single `UPPER_SNAKE_CASE` marker regex `= /(?:docs?\s*)?\[(?:doc)?(\d{1,3})\]/gi` | Verified | reasoningText.tsx:40 |
| 1.1 | `\d{1,3}` cap excludes 4-digit brackets (DD-02) | Verified | reasoningText.tsx:40 (`(\d{1,3})`) |
| 1.1 | Replace each match with ` ^$1^ ` (leading/trailing space) | Verified | reasoningText.tsx:63 |
| 1.1 | Verbatim number, no renumbering against `citations` (DR-01) | Verified | reasoningText.tsx:63 (`$1` passthrough; no citations param) |
| 1.1 | Consecutive-duplicate collapse mirroring parseAnswer `/\^(\d+)\^(?:\s*\^\1\^)+/g` | Verified | reasoningText.tsx:44,64 vs parseAnswer.tsx:26 |
| 1.1 | No `.trim()` in helper (composes after `formatReasoning`) | Verified | reasoningText.tsx:62-64 (only two `.replace`) |
| 1.1 | Pure — no React/DOM dependency | Verified | reasoningText.tsx has zero imports |
| 1.1 | `formatReasoning` untouched | Verified | reasoningText.tsx:25-33 matches research description |
| 1.2 | `[docN]` → ` ^6^ ` | Verified | reasoningText.test.tsx:66-68 |
| 1.2 | `doc[N]` → ` ^6^ `, `doc` consumed | Verified | reasoningText.test.tsx:70-74 |
| 1.2 | `docs[N]` → ` ^3^ ` | Verified | reasoningText.test.tsx:76-80 |
| 1.2 | bare `[N]` → ` ^9^ ` | Verified | reasoningText.test.tsx:82-86 |
| 1.2 | mixed `docs[3] and [9]` → both, order preserved, `and` kept | Verified | reasoningText.test.tsx:88-94 |
| 1.2 | duplicate collapse `[doc1][doc1]` → single `^1^` | Verified | reasoningText.test.tsx:96-100 |
| 1.2 | no-marker passthrough unchanged | Verified | reasoningText.test.tsx:102-106 |
| 1.2 | `[2026]` non-rewrite (DD-02 guard) | Verified | reasoningText.test.tsx:108-112 |
| 1.2 | `[note]` non-rewrite | Verified | reasoningText.test.tsx:114-118 |
| 1.3 | Run touched test file + `npx tsc -b` | Claimed green (not re-executed) | changes log: 16 in-file / 607 suite pass, tsc exit 0 |

Coverage: **9/9 plan-mandated Step 1.2 test cases present** with correct assertions; **all Step 1.1
behavioral clauses implemented exactly as specified**.

## Detailed Verification

### 1. Helper exists, exported, pure, regex/replace matches Step 1.1 spec — VERIFIED

`superscriptReasoningCitations` is declared `export function` at
[reasoningText.tsx:61](../../../../v2/src/frontend/src/pages/chat/components/reasoningText.tsx#L61-L66),
takes `text: string`, returns `string`, and contains only two chained `.replace(...)` calls — no
React import, no DOM access, no side effect. The module has **no imports at all**, so purity is
structural.

- Marker regex `REASONING_CITATION_MARKER = /(?:docs?\s*)?\[(?:doc)?(\d{1,3})\]/gi`
  ([reasoningText.tsx:40](../../../../v2/src/frontend/src/pages/chat/components/reasoningText.tsx#L40))
  is a **character-for-character match** to the Step 1.1 spec regex, covering `[docN]`, `doc[N]`,
  `docs[N]`, and bare `[N]`. The `\d{1,3}` cap (DD-02) is present.
- Replace target ` ^$1^ ` at
  [reasoningText.tsx:63](../../../../v2/src/frontend/src/pages/chat/components/reasoningText.tsx#L63)
  emits the verbatim captured number with the leading/trailing space the spec requires (DR-01, no
  renumbering — the helper has no `citations` parameter, so renumbering is structurally impossible).
- Collapse regex `CONSECUTIVE_DUPLICATE_SUP = /\^(\d+)\^(?:\s*\^\1\^)+/g`
  ([reasoningText.tsx:44](../../../../v2/src/frontend/src/pages/chat/components/reasoningText.tsx#L44))
  is identical to `parseAnswer`'s `CONSECUTIVE_DUPLICATE_SUP_PATTERN`
  ([parseAnswer.tsx:26](../../../../v2/src/frontend/src/pages/chat/components/parseAnswer.tsx#L26)),
  confirming the "mirror parseAnswer" instruction. The `^N^` token also matches the token
  `parseAnswer` produces, so both feed `remark-supersub` identically (spec intent honored).
- No `.trim()` in the helper — whitespace normalization is left to `formatReasoning`, exactly as
  Step 1.1 directs.

Manual trace of the two guard cases confirms correctness of the `\d{1,3}` cap:
- `[2026]` — the engine tries `[` then `(?:doc)?` (0) then `(\d{1,3})` greedily consumes `202`,
  needs `]` but sees `6`, backtracks to `20`/`2`, never reaches `]` — **no match, stays literal**.
- `[note]` — bracket body is non-numeric, `(?:doc)?(\d{1,3})` cannot match — **stays literal**.

### 2. `formatReasoning` NOT modified — VERIFIED

[reasoningText.tsx:25-33](../../../../v2/src/frontend/src/pages/chat/components/reasoningText.tsx#L25-L33)
retains the exact behavior the research doc records (join deltas → drop `SECTION_TITLE` bold titles →
collapse blank lines → `.trim()`). The `SECTION_TITLE` constant (line 24) is unchanged. The seven
existing `describe("formatReasoning")` tests
([reasoningText.test.tsx:18-63](../../../../v2/tests/frontend/pages/chat/components/reasoningText.test.tsx#L18-L63))
are intact and assert the same pre-existing behavior — no edits leaked into the `formatReasoning`
through-line.

### 3. Every Step 1.2 test case present and asserting correctly — VERIFIED

The new `describe("superscriptReasoningCitations")` block
([reasoningText.test.tsx:65-119](../../../../v2/tests/frontend/pages/chat/components/reasoningText.test.tsx#L65-L119))
contains all nine mandated cases. Each assertion is logically consistent with the implemented regex:

- Positive shapes assert `toContain(" ^N^ ")` (spaced token), matching the ` ^$1^ ` replacement.
- The `doc[N]` case additionally asserts `not.toContain("doc")`, correctly proving the `docs?\s*`
  prefix consumed the word.
- The mixed case asserts both tokens plus `indexOf("^3^") < indexOf("^9^")` (order) and retention of
  `and`.
- The duplicate-collapse case asserts `match(/\^1\^/g)` has length 1 — the correct positive proof
  that `[doc1][doc1]` → a single `^1^`.
- Passthrough asserts strict `toBe(...)` equality (unchanged).
- Both DD-02 guards assert `toContain("[literal]")` **and** `not.toContain("^")`.

Import is at module top
([reasoningText.test.tsx:11-15](../../../../v2/tests/frontend/pages/chat/components/reasoningText.test.tsx#L11-L15)):
`superscriptReasoningCitations` is imported from `@/pages/chat/components/reasoningText` alongside
`formatReasoning`, per Step 1.2.

### 4. CWYD Hard Rule compliance — VERIFIED (1 Info)

- **#1 (one unit/turn)**: Phase 1's production delta is one function plus its two supporting
  module-level regex constants — a single cohesive unit. Compliant.
- **#2 (test-first)**: helper landed with a full 9-case test block in the same phase. Compliant.
- **#16 (no process narrative in `src` docstring)**: the `superscriptReasoningCitations` docstring
  ([reasoningText.tsx:47-60](../../../../v2/src/frontend/src/pages/chat/components/reasoningText.tsx#L47-L60))
  and the two regex-constant comments describe *what the code is* — no phase/turn/date/"changed"
  language, no unit IDs. Compliant. (See Info-1 re: the pre-existing file header, which is out of
  Phase 1 scope.)
- **#17 (imports at module top)**: `reasoningText.tsx` has no imports; the test file's imports are
  all at the top. No lazy/in-body imports. Compliant.
- **Naming**: regex constants `REASONING_CITATION_MARKER`, `CONSECUTIVE_DUPLICATE_SUP` are
  `UPPER_SNAKE_CASE`; the function `superscriptReasoningCitations` is `camelCase`. Compliant.

### 5. Changes log accuracy for Phase 1 — VERIFIED (1 Minor)

The `reasoningText.tsx` and `reasoningText.test.tsx` "Modified" bullets in the changes log accurately
describe the delivered helper, the two named regex constants, the `\d{1,3}` cap, verbatim rewrite,
duplicate collapse, "`formatReasoning` untouched", and the "9 cases … 16 tests pass" test count
(7 `formatReasoning` + 9 new = 16 in-file). Listing `reasoningText.tsx` under **Modified** (not
Added) is correct because the file pre-existed. See Minor-1 re: the stray `_pending_` placeholder.

## Findings

### Minor

- **MINOR-1 — Changes-log "Added" section left as `_pending_` placeholder.**
  The changes log's `### Added` subsection still reads `* _pending_`
  (`.copilot-tracking/changes/2026-07-02/...changes.md`), a leftover template placeholder, even
  though the plan is marked complete. This is not incorrect (no new *files* were added; the helper
  is correctly captured under **Modified** since `reasoningText.tsx` pre-existed), but the dangling
  `_pending_` should be resolved to `* None.` for a closed plan. Cosmetic; no code impact.

### Info

- **INFO-1 — Pre-existing file header `Phase: 6 (visual polish)` uses a non-standard descriptive
  tail.** The Hard Rule #3 header at
  [reasoningText.tsx:1-3](../../../../v2/src/frontend/src/pages/chat/components/reasoningText.tsx#L1-L3)
  tails the Phase line with `(visual polish)` rather than the Phase 6 standing descriptive name
  ("Functions blueprints / modular RAG indexing pipeline"). This header is **pre-existing** (shared
  with the untouched `formatReasoning` and mirrored in `parseAnswer.tsx`), was **not** introduced by
  Phase 1, and the task explicitly notes the header is permitted. Recorded for traceability only —
  not a Phase 1 defect.
- **INFO-2 — Stale DD-01/DD-02 cross-reference note in the planning log.** The planning log's DD-02
  validation note claims the research "Accepted tradeoff" section and details Step 1.1 "cite it as
  **DD-01** … should be corrected to `DD-02`." In the current artifacts both **already** cite the
  digit-cap as **DD-02** (research: "Documented in the planning log as DD-02"; details Step 1.1:
  "Implements DD-02"). The planning-log note is therefore stale/self-inconsistent. Documentation
  cross-reference nit only; the code correctly implements the `\d{1,3}` cap regardless of numbering.

## Coverage Assessment

Phase 1 is **fully implemented** and matches the plan, details, and research at the code level. All
nine mandated test cases are present with correct assertions; the helper regex/replace logic is a
character-exact realization of the Step 1.1 specification (including the `\d{1,3}` cap and verbatim
numbering); `formatReasoning` is untouched; and all in-scope Hard Rules pass. No Critical or Major
gaps. The single Minor and two Info items are documentation-hygiene notes, none of which affect the
correctness or completeness of the Phase 1 code.

## Recommended Next Validations (not performed this session)

- [ ] Re-execute Step 1.3 gates live to confirm the changes-log green claim:
  `npx vitest run pages/chat/components/reasoningText.test.tsx` (from `v2/tests/frontend`) and
  `npx tsc -b` (from `v2/src/frontend`).
- [ ] Validate Phase 2 (Steps 2.1–2.3): `MessageList.tsx` composition of
  `superscriptReasoningCitations(formatReasoning(...))`, the `enableSupersub` prop on the reasoning
  `MarkdownContent`, the `MarkdownContent.tsx` docstring update, and the new `MessageList.test.tsx`
  `<sup>`-render assertion.
- [ ] Validate Phase 3 (Steps 3.1–3.3): full-suite green (`npm test` → 45 files / 607 tests),
  BUG-0094 `open` → `fixed` in `v2/docs/bugs.md` line 153, and the `v2/docs/worklog/2026-07-02.md`
  fixed entry (Hard Rule #19).

## Clarifying Questions

None — Phase 1 artifacts and code are self-consistent and unambiguous.
