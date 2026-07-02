---
applyTo: '.copilot-tracking/changes/2026-07-02/bug-0088-docx-and-ingestion-validation-changes.md'
---
<!-- markdownlint-disable-file -->
# Implementation Plan: BUG-0088 chunking-granularity refinement + redeploy

## Overview

Refine the Document Intelligence paragraph fallback so a pageless document (e.g. a large `.docx`) produces size-bounded grouped chunks instead of one chunk per sub-sentence DI paragraph, then redeploy the function (carrying this plus the already-landed embedder count-batching) and live-verify BUG-0088 is closed.

## Objectives

### User Requirements

* Fix BUG-0088 (`.docx` fails to ingest) end-to-end on the live deployment — Source: user request 2026-07-02.
* Fold the review's granularity + batching follow-ups into one redeploy — Source: user request 2026-07-02 (task-plan after the review).

### Derived Objectives

* Reduce the DI-fallback chunk explosion (5047 chunks for a 10-K) to a reasonable, retrieval-friendly count — Derived from: review WI + the codebase's paragraph-as-semantic-unit convention (TextParser blank-line split).
* Make one redeploy carry both fixes (count-batching + grouped fallback) to avoid a second deploy cycle — Derived from: the batching fix is code-complete but not live.

## Context Summary

### Project Files

* v2/src/functions/core/parsers/document_intelligence_parser.py - the pageless paragraph fallback to group.
* v2/src/functions/core/parsers/text_parser.py - the codebase's paragraph-as-semantic-unit convention to approximate.
* v2/src/backend/core/providers/embedders/azure_openai.py - the already-landed count-batching (≤2048) that the redeploy also ships.
* v2/tests/functions/core/parsers/test_document_intelligence_parser.py - DI parser tests to extend.

### References

* .copilot-tracking/reviews/2026-07-02/bug-0088-docx-and-ingestion-validation-plan-review.md - RV-01 (token batching) + granularity WI.
* .copilot-tracking/research/2026-07-02/bug-0088-docx-and-ingestion-validation-research.md - the primary research.
* v2/docs/bugs.md - BUG-0088, BUG-0058 (prepackage), BUG-0080 (deploy-hang repin), BUG-0049 (the fallback).

### Standards References

* .github/instructions/v2-functions-core.instructions.md - ingestion parser conventions.
* .github/instructions/v2-tests.instructions.md - test-first.
* .github/copilot-instructions.md - Hard Rules (#1/#2/#11/#16/#18/#19).

## Implementation Checklist

### [x] Implementation Phase 1: Group the DI paragraph fallback into size-bounded chunks

<!-- parallelizable: false -->

* [x] Step 1.1: Group DI paragraphs to a target character budget (PD-01) + docstring
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-chunking-refinement-details.md (Lines 12-30)
* [x] Step 1.2: Test the grouped fallback (grouped + size-bounded; long-paragraph whole; PDF page-pass unchanged)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-chunking-refinement-details.md (Lines 31-43)

### [x] Implementation Phase 2: Redeploy + live re-validation + close-out

<!-- parallelizable: false -->

* [x] Step 2.1: Regenerate the function artifact (prepackage, BUG-0058) + `azd deploy function` (watch BUG-0080)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-chunking-refinement-details.md (Lines 48-60)
* [x] Step 2.2: Live re-validate `.docx` (grouped chunks, no poison) + spot-check other types + clean up
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-chunking-refinement-details.md (Lines 61-73)
* [x] Step 2.3: Gates + mark BUG-0088 fixed in bugs.md + worklog
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-chunking-refinement-details.md (Lines 74-87)

## Planning Log

See `.copilot-tracking/plans/logs/2026-07-02/bug-0088-chunking-refinement-log.md` for the chunking-target decision (PD-01), the token-batching deferral (PD-02), and follow-on work.

## Dependencies

* Azure CLI + azd authenticated; Storage/Search data-plane RBAC (self-granted earlier this session).
* v2 uv env for gates.

## Success Criteria

* The DI fallback groups paragraphs into size-bounded chunks (a 10-K yields a reasonable count, not thousands) — Traces to: granularity WI.
* One redeploy ships both the count-batching and grouped-fallback fixes — Traces to: user "fold in" request.
* `.docx` ingests live (chunk_count > 0, listed, no poison); working file types unaffected — Traces to: BUG-0088.
* BUG-0088 marked fixed with root cause + fix; gates green; worklog updated — Traces to: Hard Rule #19 + #2.
