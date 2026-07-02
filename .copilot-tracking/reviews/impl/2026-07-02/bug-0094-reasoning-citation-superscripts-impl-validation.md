<!-- markdownlint-disable-file -->
# Implementation Quality Validation: BUG-0094 — reasoning-panel citation superscripts

Scope: full-quality. Status: **Verified (Passed)** — the Implementation Validator subagent hit a
file-access limitation in its environment and could only analyze the regex from the supplied
pattern; the reviewer completed the remaining scopes (design, consistency, tests, maintainability)
by direct file read + grep, incorporating the subagent's regex analysis.

## Finding counts

| Severity | Count |
|---|---|
| Critical | 0 |
| Major | 0 |
| Minor | 1 (optional; covered by WI-02) |
| Info | 3 |

## Findings

### IV-001 — Module-level `/g` regex `lastIndex` reuse — VERIFIED SAFE (Info)

The subagent flagged a *conditional* Critical: a module-level global regex can leak `lastIndex`
state across calls **if** consumed via `.test()`/`.exec()`. Verified by grep that
`REASONING_CITATION_MARKER` (reasoningText.tsx:38) and `CONSECUTIVE_DUPLICATE_SUP`
(reasoningText.tsx:42) are consumed **only** with `String.prototype.replace` (reasoningText.tsx:60
and :61); there are **zero** `.test(`/`.exec(` calls in the chat components. Per ECMAScript
`RegExp.prototype[Symbol.replace]`, `lastIndex` is reset to 0 at the start of every `.replace`, so a
shared module-constant global regex cannot leak state through this code path. **Not a defect.**

### IV-002 — Collapse regex duplicated across two files — Minor (optional; = WI-02)

`parseAnswer.tsx:26` (`CONSECUTIVE_DUPLICATE_SUP_PATTERN`) and `reasoningText.tsx:42`
(`CONSECUTIVE_DUPLICATE_SUP`) hold the **identical** literal `/\^(\d+)\^(?:\s*\^\1\^)+/g`. The
marker patterns differ intentionally (`DOC_MARKER_PATTERN` is canonical-only; `REASONING_CITATION_MARKER`
is the looser reasoning family), so only the *collapse* regex is duplicated. Extracting a shared leaf
module for a single 27-char literal is borderline over-engineering against CWYD's "no abstractions
for one-time operations" discipline; the duplication is already captured by planning-log **WI-02**
(unify the reasoning + answer transforms behind one shared normalizer **only if a third citation
surface appears**). Recommend leaving as-is until that trigger; no action required now.

### IV-003 — Bare `[N]` false-positive class — Info (accepted, DD-02)

Any bare 1–3 digit `[N]` in reasoning prose is superscripted regardless of whether it is a citation
(array index, footnote, literal bracket). This is the deliberate DD-02 tradeoff to satisfy the
user's `docs[3] and [9]` example; it is visual-only (a stray `<sup>`, never a broken link) and is
explicitly pinned by the `[note]` non-rewrite guard test. Consistent with the answer-body precedent.
Accepted by design.

### IV-004 — 3-digit cap slightly loose — Info

`\d{1,3}` admits up to `[999]`; realistic citation counts are far lower, and 4-digit brackets
(`[2026]`) are correctly excluded (guarded by test). Harmless; no change recommended.

## Positive assessment

* **Purity & convention** — `superscriptReasoningCitations` is a pure `(string) => string` with no
  React/DOM/imports, mirroring the `parseAnswer` / `formatReasoning` pure-transform convention.
* **Composition** — `MessageList` composes `superscriptReasoningCitations(formatReasoning(...))`
  cleanly and scopes the transform to the model-reasoning branch only (placeholder untouched); reuses
  the existing `enableSupersub` render path rather than inventing new machinery.
* **Hard Rules** — #1 (one unit/phase), #2 (test-first; 9 helper + 1 render test), #3 (Pillar/Phase
  header present), #16 (no process narrative in changed src docstrings), #17 (imports at top). No
  compile/lint errors on any of the 5 changed files.
* **Test quality** — helper tests assert token shape + order + duplicate collapse + both DD-02
  guards; the MessageList test asserts on rendered `<sup>` DOM and literal-marker absence.
