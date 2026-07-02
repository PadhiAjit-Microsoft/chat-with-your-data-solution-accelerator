<!-- markdownlint-disable-file -->
# RPI Validation: BUG-0094 — Phase 2 (Wire the superscript treatment into the reasoning panel)

- **Plan**: `.copilot-tracking/plans/2026-07-02/bug-0094-reasoning-citation-superscripts-plan.instructions.md`
- **Details**: `.copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md`
- **Research**: `.copilot-tracking/research/2026-07-02/bug-0094-reasoning-citation-superscripts-research.md`
- **Changes log**: `.copilot-tracking/changes/2026-07-02/bug-0094-reasoning-citation-superscripts-changes.md`
- **Planning log**: `.copilot-tracking/plans/logs/2026-07-02/bug-0094-reasoning-citation-superscripts-log.md`
- **Phase validated**: Phase 2 (Steps 2.1–2.3)
- **Validation date**: 2026-07-02
- **Status**: **Verified (Passed)**

## Executive Summary

Phase 2 is fully implemented and matches the plan, details, and research. All three steps
(2.1 wiring, 2.2 docstring, 2.3 test) landed as specified. Only the model-reasoning branch was
wrapped with `superscriptReasoningCitations`; the placeholder branch, the answer body render, and
all `data-testid` / `data-role` / `<details>` / `<summary>` structure are untouched. The
`MarkdownContent` docstring is present-tense and process-narrative-free. The new `MessageList`
test exists with the exact required seed string and all required assertions; existing reasoning
tests are intact. `tsc`/lint report zero errors on all three touched files.

No Critical or Major findings. Four Info notes are recorded (documentation completeness, gate
scope, unverified test-count claims, and a self-documented cross-doc numbering nit) — none block
close-out.

### Finding counts by severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| Major    | 0 |
| Minor    | 0 |
| Info     | 4 |

## Step-by-Step Coverage

### Step 2.1 — Compose helper + set `enableSupersub` on reasoning `MarkdownContent` — VERIFIED

Target file: `v2/src/frontend/src/pages/chat/components/MessageList.tsx`.

| Plan requirement | Evidence | Result |
|------------------|----------|--------|
| Import `superscriptReasoningCitations` next to `formatReasoning` | `MessageList.tsx:69` — `import { formatReasoning, superscriptReasoningCitations } from "./reasoningText";` (module top) | Pass |
| Wrap ONLY the model-reasoning branch with `superscriptReasoningCitations(formatReasoning(m.reasoning))` | `MessageList.tsx:190-196` — `m.reasoning && m.reasoning.length > 0 ? superscriptReasoningCitations(formatReasoning(m.reasoning)) : (m.reasoningPlaceholder ?? "")` | Pass |
| Placeholder branch untouched | `MessageList.tsx:196` — `: (m.reasoningPlaceholder ?? "")` unchanged | Pass |
| `enableSupersub` present on reasoning `MarkdownContent` | `MessageList.tsx:198` — `enableSupersub` prop on the `className={styles.reasoningBody}` element | Pass |
| Answer body render unchanged | `MessageList.tsx:200-204` — `<MarkdownContent className={styles.bubble} content={parsed.markdownText} enableSupersub />` matches research baseline | Pass |
| `data-testid` / `data-role` / `<details>` / `<summary>` structure untouched | `MessageList.tsx:167-186` — `data-testid={`message-${m.id}-reasoning`}`, `<summary data-streaming=…>`, `data-role={m.role}` all intact | Pass |
| Import at module top (Hard Rule #17) | `MessageList.tsx:69` is in the top import block | Pass |

### Step 2.2 — Update the `MarkdownContent` docstring — VERIFIED

Target file: `v2/src/frontend/src/pages/chat/components/MarkdownContent.tsx`.

| Plan requirement | Evidence | Result |
|------------------|----------|--------|
| Docstring states BOTH answer body and reasoning panel enable supersub | `MarkdownContent.tsx:19-23` — "Both the answer body and the reasoning panel enable it: the answer body renders the `^K^` … `parseAnswer`, and the reasoning panel renders the `^N^` … `superscriptReasoningCitations`" | Pass |
| Notes the accepted cosmetic `^..^` consequence | `MarkdownContent.tsx:23-25` — "A stray `^..^` pair in chain-of-thought therefore renders as a `<sup>` (accepted, cosmetic)." | Pass |
| No longer claims the reasoning panel leaves supersub off | Stale "leaves it off" sentence is gone; no residual claim | Pass |
| Present tense, no process narrative ("now"/"changed"/dates) | `MarkdownContent.tsx:1-26` — entirely descriptive present tense; no unit IDs, dates, or "changed"/"now" | Pass |
| Code unchanged (docstring-only edit) | `MarkdownContent.tsx:27-60` — `REMARK_PLUGINS*`, `COMPONENTS`, `MarkdownContent` unchanged | Pass |

### Step 2.3 — Extend `MessageList` tests for reasoning `<sup>` render — VERIFIED

Target file: `v2/tests/frontend/pages/chat/components/MessageList.test.tsx`.

| Plan requirement | Evidence | Result |
|------------------|----------|--------|
| ONE new `it(...)` added | `MessageList.test.tsx:230` — `it("renders reasoning citation markers as superscripts in the panel", …)` | Pass |
| Seeds finished message with mixed markers `"I checked doc[6] and docs[3] and [9]."` | `MessageList.test.tsx:231-236` — `reasoning: ["I checked doc[6] and docs[3] and [9]."]`, no `streaming` field (finished) | Pass |
| Asserts `<sup>` nodes containing 6/3/9 | `MessageList.test.tsx:252-259` — `querySelectorAll("sup")` length ≥ 1; `supText` `.toContain("6")` / `("3")` / `("9")` | Pass |
| Asserts literal `doc[6]` / `docs[3]` / `[9]` no longer present | `MessageList.test.tsx:261-263` — `details.textContent` `.not.toContain("doc[6]")` / `("docs[3]")` / `("[9]")` | Pass |
| Existing reasoning tests untouched | `MessageList.test.tsx:189` (collapsed panel), `:210` (concatenate), `:264` (section titles) — all intact; new test inserted between concatenate and section-titles | Pass |

## Hard-Rule Compliance

| Rule | Assessment | Evidence |
|------|------------|----------|
| #1 (one unit per turn) | Compliant. Phase 2 is one cohesive wiring unit (reasoning-branch render change + import) plus its docstring note and its test — no unrelated "while I'm here" edits. | `MessageList.tsx` diff is scoped to import + one render branch; `MarkdownContent.tsx` is docstring-only; test adds exactly one `it(...)`. |
| #2 (test-first) | Compliant. The wiring lands with a co-located `MessageList` test exercising the new render path. | `MessageList.test.tsx:230-264` |
| #16 (no process narrative in `src/**`) | Compliant for the Phase-2-touched src file. `MarkdownContent.tsx` docstring is present tense; no unit IDs / dates / "now"/"changed". | `MarkdownContent.tsx:1-26` — see Info-2 for gate-scope caveat and Info-1 for the untouched `MessageList.tsx` docstring. |
| #17 (all imports at module top) | Compliant. The new import is in the top import block; no lazy/in-function imports introduced. | `MessageList.tsx:69` |

## Changes Log Accuracy

The changes log entries for the three Phase-2 files are accurate against the code:

- `MessageList.tsx` — "reasoning `<details>` body's model branch now renders `superscriptReasoningCitations(formatReasoning(m.reasoning))` and the `MarkdownContent` sets `enableSupersub`. Placeholder branch, `data-testid`/`data-role`, and `<details>`/`<summary>` structure untouched." — matches `MessageList.tsx:69,190-198`.
- `MarkdownContent.tsx` — "Docstring only: … both the answer body and the reasoning panel enable supersub …" — matches `MarkdownContent.tsx:19-25`.
- `MessageList.test.tsx` — "Added one `it(...)` seeding a finished message with mixed markers … asserts the reasoning panel emits `<sup>` nodes containing `6`/`3`/`9` and no longer shows the literal `doc[6]`/`docs[3]`/`[9]` text." — matches `MessageList.test.tsx:230-263`.

(The changes log uses the word "now" in narrative prose — this is a tracking artifact, not `src/**`, so Hard Rule #16 does not apply to it.)

## Findings

### Info-1 — `MessageList.tsx` module docstring does not mention the new reasoning transform (within plan scope)

The `MessageList.tsx` module docstring (`MessageList.tsx:30-41`) still describes the reasoning body
as "formatted by `formatReasoning`" and does not mention that the model branch now also flows
through `superscriptReasoningCitations`. This is **within plan scope** — Step 2.2 deliberately
scoped the docstring update to `MarkdownContent.tsx` only — so it is not a deviation. Recorded as a
documentation-completeness note: a future touch to this file may want to add a one-line mention of
the superscript normalization. No action required for Phase 2 close-out.

### Info-2 — `test_no_process_narrative_in_src.py` does not literally scan `.tsx` files

The plan (Step 2.2) and the validation prompt reference `test_no_process_narrative_in_src.py` as
the gate the docstring "would pass". That gate walks `_SRC_ROOT.rglob("*.py")`
(`v2/tests/shared/test_no_process_narrative_in_src.py:113`) — Python files under `v2/src/` only —
so the `.tsx` docstring is not scanned by it. The docstring nonetheless satisfies the **intent** of
Hard Rule #16 (present tense, no process narrative). This is a factual clarification, not a defect.

### Info-3 — Test-count claims not re-executed (read-only validation)

The changes log asserts "34 tests pass in-file" for `MessageList.test.tsx` and "45 files / 607
tests passed" for the full suite. RPI validation is read-only analysis; these counts were not
re-run here. Static evidence (zero `tsc`/lint errors on all three touched files; new test
structurally correct; existing tests intact) is consistent with the claim, but the numeric totals
are unverified in this pass.

### Info-4 — Pre-existing DD-01/DD-02 numbering nit (Phase 1 concern, self-documented)

The planning log already records a cross-doc numbering mismatch between the log's `DD-01`/`DD-02`
and details Step 1.1's citation. This is a Phase 1 documentation concern, already flagged with a
plan-validator note in the log, and is out of Phase 2 scope. Noted here only for traceability.

## Recommended Next Validations (not performed this session)

- [ ] Phase 1 validation (Steps 1.1–1.3): verify `superscriptReasoningCitations` helper + regex
      constants + the 9 `reasoningText.test.tsx` cases, and reconcile the DD-01/DD-02 citation in
      details Step 1.1.
- [ ] Phase 3 validation (Steps 3.1–3.3): confirm `bugs.md` BUG-0094 row flipped to `fixed`, the
      `worklog/2026-07-02.md` fixed entry, and re-run `npx tsc -b` + full `npm test` to confirm the
      "607 tests passed" claim (addresses Info-3).
- [ ] Execute the touched test files live (`npx vitest run pages/chat/components/MessageList.test.tsx`)
      to confirm the new reasoning-`<sup>` assertion passes at runtime.

## Clarifying Questions

None. Phase 2 scope was unambiguous and fully evidenced by the code.
