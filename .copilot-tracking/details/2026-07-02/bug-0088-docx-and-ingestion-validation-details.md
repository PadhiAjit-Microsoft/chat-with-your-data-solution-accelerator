<!-- markdownlint-disable-file -->
# Implementation Details: BUG-0088 .docx fix + full ingestion / delete / pgvector / orchestrator validation

## Context Reference

Sources: .copilot-tracking/research/2026-07-02/bug-0088-docx-and-ingestion-validation-research.md (+ the four subagent docs under research/subagents/2026-07-02/).

## Implementation Phase 1: BUG-0088 — confirm then fix `.docx` ingestion

<!-- parallelizable: false -->

### Step 1.1: Live `.docx` ingestion probe on the current deployment

Upload a real `.docx` to the `documents` container and watch it flow through the ingestion pipeline (the method that closed BUG-0054): blob → `blob-events` → `blob_event`/`batch_push` → `cwyd-index`. Record whether it produces chunks or dead-letters.

Files:
* (live/operator action — no source edit) - Storage blob upload + queue/index observation via `az` + Search REST.

Success criteria:
* A definitive outcome: `.docx` yields `chunk_count > 0` and appears in `GET /api/admin/documents` (BUG-0088 already fixed on this fresh image), OR it dead-letters to `doc-processing-poison` (proceed to 1.2).

Context references:
* research §"Technical Scenarios → BUG-0088 fix" - the confirm-first branch.

Dependencies:
* `az` auth + Storage/Search data-plane RBAC on the deployment.

### Step 1.2: If it dead-letters — capture DI env vars + the live `AzureError`

Read the deployed Function App's `AZURE_DOCUMENT_INTELLIGENCE_API_VERSION` / `AZURE_DOCUMENT_INTELLIGENCE_MODEL_ID`, and capture the actual `AzureError` code/message from a live parse of the `.docx` (run `DocumentIntelligenceParser.parse` against the live endpoint with the deployed env values — telemetry is blind per BUG-0055).

Files:
* (live/operator action) - `az functionapp config appsettings list`; a throwaway live-parse script under `$env:TEMP` (cleaned up after).

Discrepancy references:
* Addresses the research "Potential Next Research" runtime-confirmation blocker.

Success criteria:
* The stale/incorrect DI setting (or a service content-support message) is identified with the concrete error text.

Dependencies:
* Step 1.1 outcome = dead-letter.

### Step 1.3: Apply the smallest fix + re-validate

Unset or correct the stale `AZURE_DOCUMENT_INTELLIGENCE_API_VERSION` (fall back to the `2024-11-30` source default) on the runtime; if `v2/infra/main.bicep` pins a stale value, correct it (Configuration Layer). Re-run the Step 1.1 probe → `.docx` now ingests.

Files:
* v2/infra/main.bicep - only if it pins a stale DI api-version/model (else runtime env-var change only, no source edit).

Success criteria:
* `.docx` ingests end-to-end after the change; the change is durable in Bicep if the value was provisioned.

Dependencies:
* Step 1.2 confirmed root cause.

### Step 1.4: Add a durable opt-in live `.docx` parse test + clean up

Add an opt-in integration test (skips without live DI creds) that parses a real `.docx` and asserts `> 0` chunks — the only test shape that catches a DI service content-support regression (mocked-client unit tests cannot). Clean up the Step 1.1/1.2 probe blob + chunks.

Files:
* v2/tests/functions/core/parsers/test_document_intelligence_parser_live.py - new opt-in live test (env-gated skip).

Success criteria:
* The test runs (passes with live creds; skips cleanly without), and the probe artifacts are removed (blob deleted, chunk gone, poison drained).

Dependencies:
* Step 1.1 clean OR Step 1.3 fix (a working `.docx` path to assert against — the fixed-by-redeploy branch provides it from Step 1.1, the fix branch from Step 1.3).

## Implementation Phase 2: Full file-type ingestion validation (9 types) + UI decision

<!-- parallelizable: false -->

### Step 2.1: Prepare one representative sample per supported extension

Gather/create one small file per extension: `txt`, `md`, `json`, `html`, `pdf`, `docx`, `jpeg`, `jpg`, `png` (reuse sample-data PDFs where possible; synth tiny text/markup/image files with a unique sentinel phrase each).

Files:
* (test fixtures under `$env:TEMP` — throwaway, cleaned up in 2.4) - one sample per extension.

Success criteria:
* Nine uniquely-identifiable sample files exist, each with a sentinel phrase for post-ingest search.

Dependencies:
* Phase 1 complete (`.docx` path known-good).

### Step 2.2: Ingest each; assert chunk_count > 0 + listed

Upload each sample. The UI only offers 3 types, so the 6 UI-blocked types (`md/json/html/jpeg/jpg/png`) must be dropped via the Storage container (Event-Grid path) or the admin upload API directly. For each: assert a sentinel search returns the chunk AND `GET /api/admin/documents` lists the source with `chunk_count > 0`; assert zero poison across all queues.

**Negative cases (the "silently fail" surface, per DR-04 / file-type-matrix Gaps 2/4/5):** drop an UNSUPPORTED extension (e.g. `.xlsx`) via the Event-Grid/Storage path and via reprocess-all, and record the operator-visible outcome (it hits `batch_push` `KeyError` → poison, telemetry-only — Gap 2); confirm the admin-upload extension gate returns 415 for the same file; note the DI-unconfigured 503 gate exists on admin upload but not on the reprocess/Event-Grid paths (Gap 5). Capture each observed behavior.

Files:
* (live/operator action) - per-extension positive + negative upload + index/list assertions.

Success criteria:
* All 9 supported extensions ingest (chunk_count > 0, listed); no poison on the supported set. The negative cases' actual behavior is documented (unsupported-ext poison on reprocess/Event-Grid vs 415 on admin upload). Any unexpected failure is captured with the parser + error.

Dependencies:
* Step 2.1.

### Step 2.3 (PD-01): Widen UI `ACCEPTED_EXTENSIONS` to 9 OR record 3-by-design

Per PD-01: either widen `ACCEPTED_EXTENSIONS` in `IngestData.tsx` (+ `validateFile`) to the full backend-supported 9 (with a vitest update), or record the 3-type UI as an intentional product choice in the planning log + bugs/worklog.

Files:
* v2/src/frontend/src/pages/admin/IngestData/IngestData.tsx - `ACCEPTED_EXTENSIONS` + `validateFile` (only if PD-01 = widen).
* v2/tests/frontend/pages/admin/IngestData/*.tsx - accept-list test (only if widen).

Discrepancy references:
* Addresses file-type matrix Gap 1 (UI 3-of-9).

Success criteria:
* PD-01 decided; if widened, the UI accepts all 9 and tests are green; if not, the choice is documented.

Dependencies:
* Step 2.2 (proves the 6 blocked types actually ingest, justifying the widen).

### Step 2.4: Clean up all Phase 2 test documents

Delete every Phase 2 test doc (chunks + blobs) and drain any poison, per cleanup-before-next-step.

Success criteria:
* Index/store back to baseline; queues empty; temp files removed.

## Implementation Phase 3: Delete-contract validation (chunks + blob)

<!-- parallelizable: false -->

### Step 3.1: Run the delete checklist on a representative doc

Ingest one doc, then: `DELETE /api/admin/documents/{source}` → expect `200 {deleted:N, blob_deleted:true}`; then `GET /api/admin/documents` shows the source absent; the store chunk count for that source = 0 (Search page-and-match / pgvector `SELECT COUNT(*)`); `GET /api/files/{source}` → 404; re-`DELETE` → 404 (idempotent).

Files:
* (live/operator action) - the delete-checklist assertions from research §3.

Success criteria:
* Green run: chunks removed (count 0), blob removed (`/api/files` 404), idempotent re-delete 404.

Dependencies:
* Phase 1-2 (a known-good ingest to delete).

### Step 3.2 (PD-03): Decide the orphan-gap treatment

Per PD-03: the two deletes (chunks then blob) are not transactional — a blob-delete throw after a successful chunk-delete leaves an orphan blob (endpoint 503, retry-safe). Either harden it (e.g. surface a partial-delete signal / reorder / best-effort blob delete that never masks the chunk result) or accept it as retry-safe and document the behavior.

Files:
* v2/src/backend/routers/admin.py - `delete_document_endpoint` (only if PD-03 = harden).
* v2/tests/backend/test_admin.py - delete partial-failure test (only if harden).

Discrepancy references:
* Addresses delete research Gap A (orphan blob on partial failure).

Success criteria:
* PD-03 decided; if hardened, a partial-failure test proves the behavior; if accepted, documented in the planning log.

Dependencies:
* Step 3.1.

## Implementation Phase 4: pgvector + both orchestrators validation

<!-- parallelizable: false -->

### Step 4.1 (PD-02): Stand up / point at a pgvector environment

Per PD-02: the current deployment is `cosmosdb` (+ Azure Search). pgvector/langgraph validation needs a `postgresql` environment — either a second `azd up` with `AZURE_ENV_DATABASE_TYPE=postgresql`, or a local docker stack (`docker compose -f v2/docker/docker-compose.dev.yml up`) pointed at a pgvector store.

Files:
* (operator action — infra/env) - provision or start the pgvector environment.

Success criteria:
* A reachable backend on the pgvector store with `/api/health` `pass` (database check `db_type=postgresql`).

Dependencies:
* PD-02 decision (which environment strategy).

### Step 4.2: pgvector ingestion + delete pass

On the pgvector environment, repeat the essence of Phase 2 (ingest a representative subset) + Phase 3 (delete), using pgvector SQL row-count checks (`SELECT COUNT(*) FROM documents WHERE title=$1`).

Files:
* (live/operator action) - pgvector ingest + delete assertions.

Success criteria:
* Ingest yields rows; delete removes rows (count 0) and the blob (404); no poison.

Dependencies:
* Step 4.1.

### Step 4.3: Validate all four orchestrator×store cells

For each supported cell — `langgraph`×pgvector, `langgraph`×AzureSearch, `agent_framework`×AzureSearch, `agent_framework`×pgvector — switch via `PATCH /api/admin/config`, confirm via `/config/effective` + `/status`, then assert a grounded answer with `[docN]`+filename citations, the ordered SSE frames (`reasoning` → `citation` → `answer` → `conversation`), and the out-of-domain fallback. Canonical questions: pgvector = Contoso remote-work (BUG-0065); AzureSearch = "employee benefits" (BUG-0028). For `agent_framework`×AzureSearch also assert the `knowledge_base_retrieve` tool frame + no `【…†source】` leak.

**Cell C precondition (BUG-0059, per DR-06):** before validating `agent_framework`×AzureSearch, confirm `AZURE_AI_SEARCH_CONNECTION_NAME` = the `cwyd-kb-mcp` RemoteTool connection (audience `https://search.azure.com`) — otherwise KB grounding 401s and the failure would be misattributed to the orchestrator.

Files:
* (live/operator action) - per-cell chat validation.

Success criteria:
* All four cells return grounded answers with the shared `[docN]` citation shape; the fallback fires on out-of-domain; `agent_framework`×pgvector is SERVED (not 409-rejected — asserting the ADR-0027 behavior).

Dependencies:
* Step 4.1-4.2 (pgvector env with data); the cosmosdb env for the AzureSearch cells.

### Step 4.4: Fix the stale `agent_framework`×pgvector "409-rejected" docs

Correct the stale `OrchestratorSettings.name` docstring (settings.py) that still says `agent_framework`×pgvector is 409-rejected — it is served per ADR-0027/BUG-0066. Separately, PROPOSE (do not autonomously edit) the Hard Rule #20 R3 text in `.github/copilot-instructions.md` that carries the same stale claim, per Hard Rule #0 (guidance updates proposed first, then user approval).

Files:
* v2/src/backend/core/settings.py - `OrchestratorSettings.name` docstring (code — safe to edit).
* .github/copilot-instructions.md - Hard Rule #20 R3 (guidance — PROPOSE only, await approval).

Discrepancy references:
* Addresses orchestrator research stale-doc caveat.

Success criteria:
* The code docstring reflects ADR-0027; the guidance correction is proposed to the user; no process-narrative violation.

Dependencies:
* Step 4.3 (empirically confirms the cell is served).

## Implementation Phase 5: Validation, close-out, and tracking

<!-- parallelizable: false -->

### Step 5.1: Run full gates

Backend `pytest` + `pyright --strict`; frontend `tsc -b` + `vitest`; shared AST gates (including the new env-ID gate). Fix minor issues inline; report anything larger.

Files:
* (validation only).

Success criteria:
* All gates green (or minor fixes applied); no env-ID leaks (the new gate).

### Step 5.2: Update BUG-0088 + worklog

Update BUG-0088 in `v2/docs/bugs.md` with the confirmed root cause + fix/verdict (fixed-by-redeploy or the DI api-version fix), and add the day's worklog entry (Hard Rule #19). Use placeholders for env-specific values (Hard Rule #18).

Files:
* v2/docs/bugs.md - BUG-0088 registry row + detail.
* v2/docs/worklog/2026-07-02.md - Done + Bugs entries.

Success criteria:
* BUG-0088 status reflects reality; worklog captures the validation matrix outcome; no env-ID leaks.

### Step 5.3: Report residual / blocking items

Report next steps: BUG-0055 telemetry as the enabler that would have surfaced this failure; any deferred PD; the UI-widening or orphan-gap follow-ups if deferred.

Success criteria:
* A clear residual-work list is handed to the user.

## Dependencies

* Azure CLI + azd authenticated; data-plane RBAC on Storage/Search (+ Cosmos/Postgres for the respective stores).
* v2 uv env; Node/npm.

## Success Criteria

* BUG-0088 resolved (or verdict recorded) with `.docx` ingesting live; all 9 file types validated; delete proven for chunks + blob; pgvector + all four orchestrator×store cells validated; gates green; bugs.md + worklog updated.
