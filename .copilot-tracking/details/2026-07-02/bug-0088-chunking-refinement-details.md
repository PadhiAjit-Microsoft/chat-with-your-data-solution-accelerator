<!-- markdownlint-disable-file -->
# Implementation Details: BUG-0088 chunking-granularity refinement + redeploy

## Context Reference

Sources: .copilot-tracking/reviews/2026-07-02/bug-0088-docx-and-ingestion-validation-plan-review.md (RV-01 + granularity WI), .copilot-tracking/research/2026-07-02/bug-0088-docx-and-ingestion-validation-research.md. Prior fix: AzureOpenAIEmbedder count-batching (≤2048) already landed + reviewed.

## Implementation Phase 1: Group the Document Intelligence paragraph fallback into size-bounded chunks

<!-- parallelizable: false -->

### Step 1.1: Group DI paragraphs to a target character budget

In `DocumentIntelligenceParser.parse`, change the pageless fallback from one `Chunk` per `result.paragraphs` entry to grouping consecutive paragraphs into a `Chunk` until a target character budget (`_FALLBACK_CHUNK_TARGET_CHARS`, PD-01) is reached, then emit and continue. Join grouped paragraphs with `\n\n` (paragraph-style, preserving boundaries within a chunk). Keep `index` dense; keep the page pass unchanged (paginated formats never reach the fallback, so PDFs/images keep one-chunk-per-page and never double-emit). Update BOTH the class docstring's chunking-strategy paragraph AND the in-`parse` fallback comment to describe the grouped fallback (present-tense, Hard Rule #16).

Files:
* v2/src/functions/core/parsers/document_intelligence_parser.py - the `if not chunks:` paragraph fallback (~L148-172) + the `_FALLBACK_CHUNK_TARGET_CHARS` module constant + docstring.

Discrepancy references:
* Addresses the review granularity WI (5047 one-paragraph chunks for a 10-K).

Success criteria:
* A document with many short DI paragraphs yields far fewer, size-bounded chunks (each ≤ ~target, except a single over-long paragraph which stays whole); the page pass is untouched; `index` stays dense.

Context references:
* v2/src/functions/core/parsers/text_parser.py - the codebase's paragraph-as-semantic-unit convention (blank-line split); the fallback should approximate that granularity, not DI's sub-sentence paragraphs.

Dependencies:
* PD-01 (target size).

### Step 1.2: Test the grouped fallback

Extend the DI parser tests: (a) a mocked `AnalyzeResult` with 0 pages + many short paragraphs asserts grouped, size-bounded chunks (fewer than the paragraph count, dense `index`); (b) a single paragraph longer than the target stays a whole chunk; (c) the existing PDF page-pass tests still assert one-chunk-per-page (no fallback, no double-emit).

Files:
* v2/tests/functions/core/parsers/test_document_intelligence_parser.py - grouped-fallback cases.

Success criteria:
* New cases pass; existing DI parser tests stay green; pyright 0/0 on the parser.

Dependencies:
* Step 1.1.

## Implementation Phase 2: Redeploy + live re-validation + close-out

<!-- parallelizable: false -->

### Step 2.1: Regenerate the function artifact + redeploy

Run the prepackage step (regenerate `build-functions/` from current `src/`, per BUG-0058) then `azd deploy function`. Watch for the BUG-0080 hang pattern (the `agent-framework-core` repin is in place, so the standard build should complete); if it hangs, fall back to the proven `func azure functionapp publish --no-build --python` path.

Files:
* (operator action) - prepackage + deploy.

Success criteria:
* The function redeploys with the current embedder (count-batching) + grouped DI fallback; `/api/health` on the function stays healthy.

Dependencies:
* Phase 1 (so the redeploy carries both the batching + grouping fixes).

### Step 2.2: Live re-validate `.docx` (+ spot-check)

Re-upload `MSFT_FY23Q4_10K.docx`; confirm it now ingests (chunk_count > 0, listed in `GET /api/admin/documents`, no poison) and produces a reasonable (grouped) chunk count rather than thousands. Spot-check one more DI type (a PDF already works) and one text type (`.txt`/`.md`) so the redeploy didn't regress the working paths. Clean up the test doc afterward (chunks + blob).

Files:
* (live/operator action) - upload + index/list assertions + cleanup.

Success criteria:
* `.docx` ingests end-to-end with a grouped chunk count; no poison; other types unaffected; test artifacts removed.

Dependencies:
* Step 2.1.

### Step 2.3: Gates + close-out

Run backend `pytest` + `pyright --strict` on the parser + embedder; the shared AST gates (incl. env-ID). Update BUG-0088 in `v2/docs/bugs.md` to `fixed` with the confirmed root cause (unbatched embedder + over-granular fallback) and the two-part fix (count-batching + grouped fallback), and the day's worklog. Placeholders for env-specific values (Hard Rule #18).

Files:
* v2/docs/bugs.md - BUG-0088 registry row + detail.
* v2/docs/worklog/2026-07-02.md - Done + Bugs entries.

Success criteria:
* Gates green; BUG-0088 marked fixed with root cause + fix; worklog updated; no env-ID leaks.

Dependencies:
* Step 2.2 (live confirmation).

## Dependencies

* Azure CLI + azd authenticated; Storage/Search data-plane RBAC.
* v2 uv env for gates.

## Success Criteria

* `.docx` ingests live with a grouped (not per-paragraph) chunk count; BUG-0088 closed with root cause + fix; gates green; the working file types unaffected.
