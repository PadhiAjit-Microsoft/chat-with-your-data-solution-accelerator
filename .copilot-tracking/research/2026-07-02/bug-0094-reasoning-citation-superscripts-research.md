<!-- markdownlint-disable-file -->
# Research: BUG-0094 — reasoning-panel citation markers should render as superscripts

## Scope

Make the assistant **reasoning ("Thought process") panel** render inline document-citation
markers as the same compact, tiny-indented superscript numbers used in the final answer body,
instead of showing the model's raw literal text (`doc[6]`, `[doc4]`, `docs[3] and [9]`).

Confirmed target with the user (2026-07-02): "BUG-0094 — reasoning-panel citations as superscripts".

User's original words when opening the bug: "when the thinking text is appearing we see [docx]
this should has the same format that the final response with the reference in a tiny indented number".

## Recorded bug (source of truth)

- v2/docs/bugs.md line 153 — BUG-0094 (frontend, medium, open, 2026-07-02).
  - Root cause recorded: `MessageList.tsx` renders the reasoning body via
    `<MarkdownContent content={formatReasoning(m.reasoning)} />` **without** `enableSupersub` and
    **without** `parseAnswer`, whereas the answer body renders
    `<MarkdownContent content={parsed.markdownText} enableSupersub />`.
  - Fix direction recorded: extend the `parseAnswer` / `remark-supersub` marker-to-superscript
    rendering to the reasoning body, handling the `doc[N]` shape.
  - Distinct from BUG-0043 (stripped native KB `【4:0†source】` markers from reasoning).

## Precedent — the answer body already solves this (BUG-0016)

- v2/docs/bugs.md line 74 — BUG-0016 (fixed 2026-06-15). Inline `[docN]` rendered as literal text;
  fixed with `parseAnswer` (rewrites `[docN]` → `^K^`, deduped + renumbered) + a `MarkdownContent`
  `enableSupersub` prop that runs `remark-supersub` so the answer body shows visual-only `<sup>`
  citation numbers.

## Key code findings (read directly)

### v2/src/frontend/src/pages/chat/components/MessageList.tsx

- Reasoning `<details>` panel renders its body through:
  ```tsx
  <MarkdownContent
    className={styles.reasoningBody}
    content={
      m.reasoning && m.reasoning.length > 0
        ? formatReasoning(m.reasoning)
        : (m.reasoningPlaceholder ?? "")
    }
  />
  ```
  (inside the `<details data-testid={`message-${m.id}-reasoning`}>` block). **No `enableSupersub`.**
- Answer body renders through:
  ```tsx
  <MarkdownContent className={styles.bubble} content={parsed.markdownText} enableSupersub />
  ```
  where `parsed = parseAnswer(m.content, m.citations)`.
- So the only structural difference is (a) the reasoning body never runs a marker→`^K^` transform,
  and (b) it never sets `enableSupersub`.

### v2/src/frontend/src/pages/chat/components/parseAnswer.tsx

- `DOC_MARKER_PATTERN = /\[doc(\d+)\]/g` — matches only the canonical `[docN]` shape.
- `CONSECUTIVE_DUPLICATE_SUP_PATTERN = /\^(\d+)\^(?:\s*\^\1\^)+/g` — collapses repeated adjacent
  superscripts to one; reusable for the reasoning transform.
- `parseAnswer` **renumbers** markers `1..K` against the message `citations` array and returns the
  referenced subset for the clickable `CitationPanel`. The reasoning panel is chain-of-thought and
  is **not** a clickable citation surface, so renumbering is not required there — a visual-only
  superscript of the number the model wrote is what the user asked for ("a tiny indented number").

### v2/src/frontend/src/pages/chat/components/reasoningText.tsx

- `formatReasoning(parts: string[]): string` — joins the reasoning deltas, strips model bold section
  titles (`SECTION_TITLE`), collapses blank lines. It does **not** touch citation markers.
- Natural home for a new sibling pure helper `superscriptReasoningCitations(text: string): string`.

### v2/src/frontend/src/pages/chat/components/MarkdownContent.tsx

- `enableSupersub` prop toggles `remark-supersub` in the remark pipeline. `^text^` → `<sup>`,
  `~text~` → `<sub>`. Raw HTML is escaped (`rehype-raw` deliberately omitted), so the ONLY way to
  produce a `<sup>` is the `^..^` token path — the reasoning transform must emit `^N^`, same as the
  answer.
- Docstring currently states: "the reasoning panel leaves it off so stray `^`/`~` in
  chain-of-thought stays literal." This becomes stale once supersub is enabled on the reasoning
  panel and must be updated to describe the new behavior.

## Reasoning marker shapes to normalize (from the recorded bug + user examples)

| Observed in reasoning | Meaning        | Target |
|-----------------------|----------------|--------|
| `[doc6]`              | canonical      | `^6^`  |
| `doc[6]`              | word + bracket | `^6^`  |
| `docs[3]`             | plural + brkt  | `^3^`  |
| `[9]` (bare)          | continuation   | `^9^`  |
| `docs[3] and [9]`     | mixed          | `^3^ and ^9^` |

A single regex family covers all: an optional `docs?` prefix (with optional whitespace) + an
optional `doc` inside the brackets + `\d{1,3}` digits. Verbatim number, no renumbering.

## Known constraints

- CWYD Hard Rule #1 — one unit (method/class) per implementation turn; #2 — test-first.
- Hard Rule #11/#16/#17 — imports-at-top, no process narrative in `src/**` docstrings.
- Naming: TS `camelCase` functions, `UPPER_SNAKE_CASE` module constants (regex patterns).
- The transform is a **pure string function** (mirrors `parseAnswer` / `formatReasoning` style) —
  no React, no side effects — so it is unit-testable in isolation.
- Frontend tooling: tests via `npx vitest run` from `v2/tests/frontend/`; typecheck via
  `npx tsc -b` from `v2/src/frontend/`; full suite `npm test` from `v2/`.

## Accepted tradeoff (design risk)

Matching **bare** `[N]` is required to satisfy the user's `docs[3] and [9]` example, but a genuine
numeric bracket in chain-of-thought (e.g. a literal `[2026]` year) could be turned into a
superscript. Mitigation: cap digits to `\d{1,3}` (citations never realistically exceed a few
dozen; 4-digit years like `[2026]` are excluded). This is a visual-only panel, so a false positive
is cosmetic, never a broken link. Documented in the planning log as DD-02.

## Recommended fix (two units)

1. **Unit 1** — add `superscriptReasoningCitations(text: string): string` to `reasoningText.tsx`
   (pure), rewriting the marker family → ` ^N^ ` and collapsing consecutive duplicates; test-first
   in `reasoningText.test.tsx`.
2. **Unit 2** — in `MessageList.tsx`, compose the helper over the model-reasoning branch and set
   `enableSupersub` on the reasoning `MarkdownContent`; update the `MarkdownContent` docstring note;
   extend `MessageList.test.tsx` to assert the reasoning panel emits `<sup>` and no longer shows
   literal `doc[N]` text.
3. **Close-out** — mark BUG-0094 fixed in `v2/docs/bugs.md` + append to the day worklog
   (Hard Rule #19).
