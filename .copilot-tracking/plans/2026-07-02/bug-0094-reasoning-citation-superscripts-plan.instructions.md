---
applyTo: '.copilot-tracking/changes/2026-07-02/bug-0094-reasoning-citation-superscripts-changes.md'
---
<!-- markdownlint-disable-file -->
# Implementation Plan: BUG-0094 — reasoning-panel citation markers as superscripts

## Overview

Render inline document-citation markers in the assistant reasoning ("Thought process") panel as the
same compact superscript numbers used in the final answer, by normalizing the reasoning marker
shapes (`[docN]`, `doc[N]`, `docs[N]`, bare `[N]`) into `^N^` tokens and enabling `remark-supersub`
on the reasoning body.

## Objectives

### User Requirements

* Reasoning-panel citations must show as "the same format that the final response with the reference
  in a tiny indented number" instead of raw `[docx]` text — Source: conversation (bug opened
  2026-07-02) and confirmed plan target 2026-07-02.

### Derived Objectives

* Keep the transform a **pure, isolated string helper** mirroring `parseAnswer` / `formatReasoning`
  — Derived from: existing frontend convention that citation/reasoning transforms are pure and
  unit-tested separately from the React render (parseAnswer.tsx, reasoningText.tsx).
* Use the number the model wrote **verbatim** (no renumbering against `citations`) — Derived from:
  the reasoning panel is chain-of-thought and is not a clickable citation surface, so the user's ask
  is visual-only; renumbering would risk mismatching the model's free-form numbering.
* Keep BUG-0094 registry + worklog in lockstep with the fix — Derived from: CWYD Hard Rule #19
  (durable file-based tracking).

## Context Summary

### Project Files

* v2/src/frontend/src/pages/chat/components/reasoningText.tsx - Home of `formatReasoning`; add the
  new pure `superscriptReasoningCitations` helper here as a sibling.
* v2/src/frontend/src/pages/chat/components/MessageList.tsx - Renders the reasoning `<details>` body
  via `<MarkdownContent content={formatReasoning(...)} />` with no `enableSupersub`; wiring target.
* v2/src/frontend/src/pages/chat/components/parseAnswer.tsx - Precedent for the marker→`^K^` rewrite
  and the `CONSECUTIVE_DUPLICATE_SUP_PATTERN` collapse regex (reuse the collapse approach).
* v2/src/frontend/src/pages/chat/components/MarkdownContent.tsx - `enableSupersub` prop plumbing;
  docstring note about the reasoning panel leaving supersub off must be updated.
* v2/tests/frontend/pages/chat/components/reasoningText.test.tsx - Extend with helper unit tests.
* v2/tests/frontend/pages/chat/components/MessageList.test.tsx - Extend with a reasoning-panel
  `<sup>` render assertion.
* v2/docs/bugs.md - BUG-0094 row (line 153) flips to fixed on close-out.
* v2/docs/worklog/2026-07-02.md - Append the fixed entry.

### References

* .copilot-tracking/research/2026-07-02/bug-0094-reasoning-citation-superscripts-research.md -
  Full findings, marker-shape table, and accepted tradeoff.
* v2/docs/bugs.md (line 74) - BUG-0016, the answer-body precedent this fix mirrors.

### Standards References

* .github/copilot-instructions.md — Hard Rules #1 (one unit/turn), #2 (test-first), #11/#16/#17
  (code hygiene), #19 (durable tracking).
* .github/instructions/v2-frontend.instructions.md — CWYD v2 React/Vite frontend conventions.
* .github/instructions/v2-tests.instructions.md — test-first contract, vitest conventions.

## Implementation Checklist

### [ ] Implementation Phase 1: Pure reasoning-citation superscript helper

<!-- parallelizable: false -->

* [ ] Step 1.1: Add `superscriptReasoningCitations(text: string): string` to reasoningText.tsx
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 12-44)
* [ ] Step 1.2: Add helper unit tests covering every marker shape + no-marker passthrough
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 46-70)
* [ ] Step 1.3: Run the reasoningText test file and typecheck
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 72-80)

### [ ] Implementation Phase 2: Wire the superscript treatment into the reasoning panel

<!-- parallelizable: false -->

* [ ] Step 2.1: Compose the helper + set `enableSupersub` on the reasoning `MarkdownContent`
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 84-118)
* [ ] Step 2.2: Update the MarkdownContent docstring note about the reasoning panel
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 120-134)
* [ ] Step 2.3: Extend MessageList tests to assert reasoning `<sup>` render + no literal marker
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 136-160)

### [ ] Implementation Phase 3: Validation and close-out

<!-- parallelizable: false -->

* [ ] Step 3.1: Run full frontend validation (typecheck + full vitest + shared gates)
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 164-182)
* [ ] Step 3.2: Mark BUG-0094 fixed in bugs.md and append the worklog entry
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 184-200)
* [ ] Step 3.3: Report blocking issues, if any, and defer the live re-verify follow-on
  * Details: .copilot-tracking/details/2026-07-02/bug-0094-reasoning-citation-superscripts-details.md (Lines 202-214)

## Planning Log

See .copilot-tracking/plans/logs/2026-07-02/bug-0094-reasoning-citation-superscripts-log.md for
discrepancy tracking, implementation paths considered, and suggested follow-on work.

## Dependencies

* Node/npm frontend toolchain under v2/src/frontend (react-markdown, remark-supersub already
  installed and used by the answer body).
* Vitest + jsdom test harness under v2/tests/frontend.

## Success Criteria

* The reasoning panel renders `doc[N]` / `docs[N]` / `[docN]` / bare `[N]` as `<sup>N</sup>`
  superscripts, with no literal bracketed markers left in the panel text — Traces to: user
  requirement (tiny indented number format).
* `superscriptReasoningCitations` is a pure, independently unit-tested helper — Traces to: derived
  objective (pure transform convention).
* `npx tsc -b` is clean and the full frontend vitest suite is green — Traces to: CWYD "every phase
  ends green" + test-first contract.
* BUG-0094 is marked fixed in bugs.md and recorded in the 2026-07-02 worklog — Traces to: Hard
  Rule #19.
