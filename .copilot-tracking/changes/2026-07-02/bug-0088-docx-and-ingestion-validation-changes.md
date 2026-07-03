<!-- markdownlint-disable-file -->
# Release Changes: BUG-0088 .docx fix + full ingestion / delete / pgvector / orchestrator validation

**Related Plan**: bug-0088-docx-and-ingestion-validation-plan.instructions.md
**Implementation Date**: 2026-07-02

## Summary

Confirm + fix BUG-0088 (`.docx` dead-letters at Document Intelligence while PDFs ingest), then validate the full ingestion surface: all supported file types, the delete contract (chunks + blob), pgvector integration, and both orchestrators.

## Changes

### Added

* (pending)

### Modified

* v2/src/backend/core/providers/embedders/azure_openai.py - Phase 1 (BUG-0088 fix): batch `embed` into `_MAX_EMBED_INPUTS` (2048)-sized requests and concatenate the results, so a document whose chunk count exceeds the Azure OpenAI input-array cap no longer 400s. pyright 0/0.
* v2/tests/backend/core/providers/embedders/test_azure_openai.py - Phase 1: added `test_embed_batches_inputs_over_the_array_cap` (cap patched to 2, 5 chunks -> 3 batched calls + concatenated vectors). 8 embedder tests pass.
* v2/src/functions/core/parsers/document_intelligence_parser.py - chunking-refinement Phase 1 (BUG-0088 granularity): the pageless paragraph fallback now groups consecutive `result.paragraphs` into chunks of up to `_FALLBACK_CHUNK_TARGET_CHARS` (2000) characters (joined with a blank line; a single over-long paragraph stays whole), instead of one chunk per DI paragraph -- so a large `.docx` yields low hundreds of chunks, not thousands. Docstring + in-parse comment updated. pyright 0/0.
* v2/tests/functions/core/parsers/test_document_intelligence_parser.py - chunking-refinement Phase 1: updated the two fallback tests to the grouped shape and added `test_parse_groups_paragraphs_to_target_char_budget` + `test_parse_keeps_over_long_paragraph_as_whole_chunk`. 30 parser tests pass.

### Removed

* (pending)

## Additional or Deviating Changes

* **Root-cause diagnosis (Phase 1 Steps 1.1-1.2, live + local).** The confirm-first probe uploaded `MSFT_FY23Q4_10K.docx` to the live deployment; it dead-lettered to `doc-processing-poison` after 5 retries (~5.5 min). A local probe against the SAME live DI endpoint proved current-source `DocumentIntelligenceParser.parse` produces **5047 chunks** (the paragraph fallback works) with no throw; a local parse+embed then proved `AzureOpenAIEmbedder.embed` raises `openai.BadRequestError: array length must be 2048 or less` on the 5047-input request. So BUG-0088 is an **unbatched-embedder** defect (large chunk sets exceed the Azure OpenAI input cap), NOT the DI api-version issue (the deployed function uses the correct `2024-11-30` default with no override) nor a stale image (the parser fallback is present). Both original plan hypotheses were superseded by the live evidence (planning log DD-04).
* **Deviation from plan Step 1.3.** The fix is a code change to the embedder (batching), not an env/Bicep DI-api-version change; Step 1.3's api-version branch is not needed.
* **Redeployed + live-verified 2026-07-02.** `azd deploy function` (3m16s, no BUG-0080 hang) shipped both fixes (embedder count-batching + grouped DI fallback) to the function Container App. Live: `MSFT_FY23Q4_10K.docx` ingests **198 chunks** (`cwyd-index` 263 → 461), zero poison — vs the prior 5047-chunk dead-letter; deleting the blob removed all 198 chunks (→ 263) + the blob via the BlobDeleted path. Close-out gate sweep 1147 passed / 1 skip. Prepackage is N/A here — the Container App function builds from source via `Dockerfile.functions` (the sibling plan's prepackage step applied to the retired Flex Consumption path).
* All probe artifacts cleaned up (probe blob deleted, stale poison message drained, temp scripts removed; all queues at 0).

## Release Summary

BUG-0088 (`.docx` dead-letters at ingestion) is **fixed and live-verified**. Root cause: an unbatched embedder plus an over-granular Document Intelligence paragraph fallback — a 10-K `.docx` split into 5047 one-paragraph chunks and the embedder sent them all in one request, exceeding Azure OpenAI's 2048-input cap (`openai.BadRequestError`) → dead-letter. Two-part code fix: (1) `AzureOpenAIEmbedder.embed` batches into 2048-input requests; (2) the DI fallback groups paragraphs into ~2000-char chunks (10-K → 198 chunks). Redeployed the function Container App and verified live: the `.docx` ingests 198 chunks with zero poison; delete removes chunks + blob. Files: `azure_openai.py` + its test (batching), `document_intelligence_parser.py` + its test (grouping). Follow-ups (deferred): token-aware batching (WI-01), the sibling plan's Phases 2-5 validation matrix (WI-02), sibling Step 1.4 doc reconcile (WI-03).
