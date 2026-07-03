---
applyTo: '.copilot-tracking/changes/2026-07-02/bug-0088-docx-and-ingestion-validation-changes.md'
---
<!-- markdownlint-disable-file -->
# Implementation Plan: BUG-0088 .docx fix + full ingestion / delete / pgvector / orchestrator validation

## Overview

Confirm and fix BUG-0088 (a `.docx` upload dead-letters at Document Intelligence while PDFs ingest), then run a comprehensive live validation of the ingestion surface: every supported upload file type, the delete contract (index/store chunks AND the storage blob), PostgreSQL/pgvector integration, and both orchestrators (`agent_framework` + `langgraph`).

## Objectives

### User Requirements

* Fix BUG-0088 (`.docx` fails to ingest) — Source: user request 2026-07-02 ("let fix the bug 88").
* Review + validate all available upload file types — Source: user request 2026-07-02.
* Validate that deleting a file removes its chunks AND deletes the blob from storage — Source: user request 2026-07-02.
* Validate PostgreSQL/pgvector integration — Source: user request 2026-07-02.
* Validate both `agent_framework` and `langgraph` orchestrators — Source: user request 2026-07-02.

### Derived Objectives

* Confirm-before-fix: establish whether BUG-0088 still reproduces on the fresh current deployment before changing code — Derived from: research finding that the source default is already correct and the current image is fresh (may be fixed-by-redeploy).
* Add a durable regression guard for Office-format parsing — Derived from: research finding that mocked-client unit tests cannot catch a DI service content-support gap.
* Surface + decide the UI file-type gap (UI offers 3 of 9 supported types) — Derived from: file-type matrix research (Gap 1).
* Correct the stale `agent_framework`×pgvector "409-rejected" documentation — Derived from: orchestrator research (ADR-0027 superseded ADR-0022; docstring + Hard Rule #20 R3 are stale).

## Context Summary

### Project Files

* v2/src/functions/core/parsers/document_intelligence_parser.py - the `.docx`/PDF/image parser; DI call + BUG-0049 paragraph fallback.
* v2/src/functions/batch_push/handler.py - queue handler; treats 0 chunks as a warning (not a raise) — the reconciliation that proves BUG-0088 is a throw.
* v2/src/functions/batch_push/blueprint.py - parser routing (`registry.get(parser_key_for_path(...))`, no fallback).
* v2/src/backend/core/settings.py - `AZURE_DOCUMENT_INTELLIGENCE_*` (api-version / model id) settings; `OrchestratorSettings.name` (stale docstring); `DatabaseSettings._enforce_mode_consistency`.
* v2/src/backend/routers/admin.py - `DELETE /api/admin/documents/{source}` (chunks + blob).
* v2/src/backend/services/files.py - `delete_document` (blob deletion).
* v2/src/backend/core/providers/search/pgvector.py - pgvector schema / search / delete.
* v2/src/frontend/src/pages/admin/IngestData/IngestData.tsx - UI `ACCEPTED_EXTENSIONS` (3 of 9).
* v2/infra/main.bicep - DI env-var wiring (if any) + per-db-type orchestrator default.

### References

* .copilot-tracking/research/2026-07-02/bug-0088-docx-and-ingestion-validation-research.md - primary research.
* .copilot-tracking/research/subagents/2026-07-02/bug-0088-docx-root-cause-research.md - root-cause detail.
* .copilot-tracking/research/subagents/2026-07-02/file-type-matrix-research.md - file-type matrix.
* .copilot-tracking/research/subagents/2026-07-02/delete-path-and-pgvector-research.md - delete + pgvector.
* .copilot-tracking/research/subagents/2026-07-02/orchestrators-store-matrix-research.md - orchestrator matrix.
* v2/docs/bugs.md - BUG-0088, BUG-0049, BUG-0057, BUG-0058, BUG-0055, BUG-0073, BUG-0048, BUG-0065, BUG-0064, BUG-0066.

### Standards References

* .github/copilot-instructions.md - Hard Rules (#0 sync guidance, #1 one-unit, #2 test-first, #10 structural, #18 no env IDs, #19 durable tracking, #20 citations).
* .github/instructions/v2-functions.instructions.md - Functions blueprint conventions.
* .github/instructions/v2-tests.instructions.md - test-first + gate conventions.

## Implementation Checklist

### [ ] Implementation Phase 1: BUG-0088 — confirm then fix `.docx` ingestion

<!-- parallelizable: false -->

* [x] Step 1.1: Live `.docx` ingestion probe on the current deployment
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 12-27)
* [x] Step 1.2: If it dead-letters — capture DI env vars + the live `AzureError` (root-cause confirm)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 28-43)
* [ ] Step 1.3: Apply the smallest fix (unset/correct stale DI api-version; Bicep if pinned) + re-validate
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 44-56)
* [x] Step 1.4: Add a durable opt-in live `.docx` parse test (asserts > 0 chunks) + clean up probe artifacts
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 57-69)

### [ ] Implementation Phase 2: Full file-type ingestion validation (9 types) + UI decision

<!-- parallelizable: false -->

* [ ] Step 2.1: Prepare one representative sample per supported extension (txt/md/json/html/pdf/docx/jpeg/jpg/png)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 74-86)
* [ ] Step 2.2: Ingest each (Storage/Event-Grid + admin API for the 6 UI-blocked types); assert chunk_count > 0 + listed
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 87-101)
* [ ] Step 2.3 (PD-01): Widen UI `ACCEPTED_EXTENSIONS` to the full 9 OR record 3-by-design
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 102-118)
* [ ] Step 2.4: Clean up all Phase 2 test documents (chunks + blobs)

### [ ] Implementation Phase 3: Delete-contract validation (chunks + blob)

<!-- parallelizable: false -->

* [ ] Step 3.1: Run the delete checklist on a representative doc (DELETE → {deleted:N, blob_deleted:true} → absent → 404)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 130-142)
* [ ] Step 3.2 (PD-03): Decide the orphan-gap treatment (harden the non-transactional two-delete, or accept as retry-safe + document)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 143-159)

### [ ] Implementation Phase 4: pgvector + both orchestrators validation

<!-- parallelizable: false -->

* [ ] Step 4.1 (PD-02): Stand up / point at a pgvector environment (postgresql `azd up` OR local docker stack)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 164-176)
* [ ] Step 4.2: pgvector ingestion + delete pass (SQL row-count checks; repeat Phase 2/3 essence)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 177-189)
* [ ] Step 4.3: Validate all four orchestrator×store cells (switch via PATCH /api/admin/config; grounding + SSE frames + fallback)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 190-204)
* [ ] Step 4.4: Fix the stale `OrchestratorSettings.name` docstring; propose the Hard Rule #20 R3 guidance correction (Hard Rule #0)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 205-221)

### [ ] Implementation Phase 5: Validation, close-out, and tracking

<!-- parallelizable: false -->

* [ ] Step 5.1: Run full gates (backend pytest + pyright, frontend tsc + vitest, shared AST gates incl. the env-ID gate)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 226-234)
* [ ] Step 5.2: Update BUG-0088 in v2/docs/bugs.md (root cause + fix/verdict) + the day's worklog (Hard Rule #19)
  * Details: .copilot-tracking/details/2026-07-02/bug-0088-docx-and-ingestion-validation-details.md (Lines 235-246)
* [ ] Step 5.3: Report residual/blocking items (BUG-0055 telemetry enabler; any deferred PD) as next steps

## Planning Log

See `.copilot-tracking/plans/logs/2026-07-02/bug-0088-docx-and-ingestion-validation-log.md` for discrepancy tracking, implementation paths considered, planning decisions (PD-01..03), and suggested follow-on work.

## Dependencies

* Azure CLI (`az`) + `azd` authenticated to the deployment subscription (live validation).
* The v2 uv env (`v2/.venv`) for pytest gates; Node/npm for the frontend edit + gates.
* A pgvector environment for Phase 4 (postgresql `azd up` or local docker stack) — see PD-02.
* Storage Queue + Blob + Search Index data-plane RBAC on the deployment for live checks (self-granted during the BUG-0054 work; re-grant if removed).

## Success Criteria

* `.docx` ingests end-to-end on the live deployment (chunk_count > 0, listed) — Traces to: BUG-0088 / user requirement.
* All 9 supported file types ingest successfully; the UI gap is decided (widened or documented) — Traces to: "review all file types" + Gap 1.
* Deleting a document is proven to remove both the chunks and the blob (index/pgvector count 0 + `GET /api/files/{source}` 404) — Traces to: delete requirement.
* pgvector ingestion + delete + grounded chat validated on a postgresql environment — Traces to: pgvector requirement.
* All four orchestrator×store cells validated (grounded answer + `[docN]` citations + fallback) — Traces to: both-orchestrators requirement.
* BUG-0088 closed in bugs.md with root cause + fix/verdict; worklog updated; all gates green — Traces to: Hard Rule #19 + #2.
