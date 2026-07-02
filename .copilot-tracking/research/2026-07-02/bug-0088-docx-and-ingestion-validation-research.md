<!-- markdownlint-disable-file -->
# Task Research: BUG-0088 .docx ingestion + full ingestion/delete/pgvector/orchestrator validation

Fix BUG-0088 (a `.docx` upload produces 0 chunks and dead-letters while PDFs ingest fine) and, around that fix, establish a comprehensive validation of the ingestion + retrieval surface: every uploadable file type, the delete contract (chunks removed from the index/store AND the blob removed from storage), PostgreSQL/pgvector integration, and both orchestrators (`agent_framework` + `langgraph`).

## Task Implementation Requests

* Fix **BUG-0088** — `.docx` (and by extension other Office / non-PDF formats) fails to ingest (0 chunks, dead-letters to poison) while PDFs succeed.
* Review + validate **all available upload file types** — enumerate every supported type, confirm each parses → embeds → indexes.
* Validate the **delete contract** — deleting a document removes its chunks from the search index / pgvector store AND deletes the underlying blob from Storage.
* Validate **PostgreSQL / pgvector integration** — ingestion + retrieval on the `postgresql` profile.
* Validate **both orchestrators** — `agent_framework` and `langgraph` — including which store each is valid against.

## Scope and Success Criteria

* Scope: v2 ingestion pipeline (`v2/src/functions/**`), backend delete/admin/search/db providers (`v2/src/backend/**`), the two orchestrators, and the live deployment behavior. Excludes v1 (`code/`).
* Assumptions:
  * The active cloud deployment is `<AZD_ENV_NAME>` (suffix `<SUFFIX>`), `cosmosdb` + `agent_framework`, function hosted as `ca-func-<SUFFIX>` (Container App), deployed today.
  * pgvector + langgraph validation may require a `postgresql`-profile deployment or a local stack pointed at a pgvector store.
* Success Criteria:
  * BUG-0088 root cause is determined with file:line evidence and a recommended fix (or a "stale-deploy, already fixed in source" verdict with a validation plan).
  * A complete supported-file-type matrix (extension → parser → status) exists.
  * The delete flow (chunks + blob) is documented with the exact code path and a validation checklist.
  * The orchestrator × store compatibility matrix is documented with per-cell validation steps.

## Outline

1. BUG-0088 root cause (source-level + deployed-artifact analysis).
2. Supported file-type matrix (parsers + registered extensions).
3. Delete contract (index/pgvector chunk removal + blob removal).
4. pgvector integration points + validation.
5. Orchestrator × store compatibility + validation.
6. Consolidated fix + validation plan (the hand-off to implementation).

## Potential Next Research

* **Live confirmation of BUG-0088 root cause** (implement-phase / operator action — the single blocker to a definitive fix):
  * Reasoning: the source path is correct; the failure is a runtime throw from Document Intelligence on `.docx` bytes. Only a live capture distinguishes the hypotheses.
  * Reference: `az functionapp config appsettings list` for `AZURE_DOCUMENT_INTELLIGENCE_API_VERSION` / `AZURE_DOCUMENT_INTELLIGENCE_MODEL_ID`; a live parse of the real `.docx` capturing the `AzureError` code/message.
* **Confirm the deployed function image is current** (BUG-0058 stale-artifact theme) — the fresh `<AZD_ENV_NAME>` deploy is today's image, but verify it carries the paragraph fallback + the `2024-11-30` DI default.
* **Whether `v2/infra/main.bicep` wires the two `AZURE_DOCUMENT_INTELLIGENCE_*` env vars** — if a stale value can be pinned at provision time.

## Research Executed

### Subagent investigations (dispatched 2026-07-02)

* Subagent 1 — BUG-0088 .docx root cause.
* Subagent 2 — supported file-type matrix.
* Subagent 3 — delete path (chunks + blob) + pgvector integration.
* Subagent 4 — orchestrators (agent_framework + langgraph) × store.

## Key Discoveries

### 1. BUG-0088 root cause — a Document Intelligence THROW on `.docx`, not a stale BUG-0049

**Decisive reconciliation:** the `batch_push` handler treats **0 chunks as a warning + `return []` (silent success), never a raise** ([handler.py](v2/src/functions/batch_push/handler.py) ~L90-101; docstring: "raising would poison-loop it forever"). BUG-0049 (the pageless-Office `pages[*].lines` empty case) was therefore **silent** — it returned `200` and never dead-lettered. BUG-0088 **dead-letters**, which requires a **throw**. So BUG-0088 is a *different defect class* than BUG-0049, and the paragraph fallback (BUG-0049's fix) **is present in current source** ([document_intelligence_parser.py](v2/src/functions/core/parsers/document_intelligence_parser.py) ~L148-172, unit-tested).

**The only divergence between `.docx` (fails) and `.pdf` (works) is the bytes handed to `begin_analyze_document`** — same parser class, model, api-version, embedder, and search push. So the throw is inside the Document Intelligence call → `AzureError` re-raised (~L112-128) → `@log_queue_errors` re-raises → 5 retries → poison.

**Top hypothesis:** Office-format support (DOCX/PPTX/XLSX/HTML) for `prebuilt-layout` requires api-version **`2024-02-29-preview`+ (GA `2024-11-30`)**; PDF parses at *every* version. The source default is correct (`api_version=2024-11-30`, `model_id=prebuilt-layout`, [settings.py](v2/src/backend/core/settings.py#L497)), **but both are operator-pinnable via `AZURE_DOCUMENT_INTELLIGENCE_*` env vars** — a stale env pin (or a deployed image predating the GA default) makes `.docx` throw while `.pdf` succeeds. Unit tests can't catch it (the DI client is always mocked, so Office-format service handling is never exercised).

**Confirmation needs a runtime action** (BUG-0055/0089 telemetry is blind, so reprocess alone won't reveal it): read the deployed Function App's two DI env vars + capture the live `AzureError`. **Caveat:** the current `<AZD_ENV_NAME>` deployment is a *fresh today* image — BUG-0088 may already be resolved there; the first validation step is simply to upload the `.docx` and observe.

### 2. Supported file-type matrix — 9 backend types, only 3 reachable via the UI

Authoritative supported set = the 9 `ParserKey` extensions ([base.py](v2/src/backend/core/providers/parsers/base.py) L30-38); routing is `registry.get(parser_key_for_path(filename))` ([batch_push/blueprint.py](v2/src/functions/batch_push/blueprint.py) L118-120) with **no fallback parser** — an unregistered extension raises `KeyError` → poison.

| Extension | Parser | Mechanism |
|---|---|---|
| `txt`, `md`, `json` | `TextParser` | plain UTF-8, paragraph chunking |
| `html` | `HtmlParser` | bs4 (strips script/style/noscript) |
| `pdf`, `docx`, `jpeg`, `jpg`, `png` | `DocumentIntelligenceParser` | Azure DI `prebuilt-layout` |

**Gaps (where uploads silently fail):**
1. **UI regression (biggest):** the frontend hardcodes `ACCEPTED_EXTENSIONS = [".pdf", ".docx", ".txt"]` ([IngestData.tsx](v2/src/frontend/src/pages/admin/IngestData/IngestData.tsx) L55) and client-rejects the other 6 supported types — `md/json/html/jpeg/jpg/png` are backend-supported but **UI-unreachable**.
2. **Silent poison:** admin upload validates the extension (415), but **reprocess-all + Event-Grid blob-drop do NOT** — an unregistered-extension blob poisons via `KeyError`.
3. **URL entry-point divergence:** Functions `/api/add_url` defaults ext-less URLs to `txt` (raw markup) vs the admin route's `html` (clean text).
4. DI docstring claims xlsx/pptx but they aren't registered → 415.

**v1 parity:** 9/9 at the parser/backend level (guarded by `test_supported_extensions.py`, BUG-0074), but only **3/9 at the UI**.

### 3. Delete contract — store-agnostic two-part removal, with one orphan gap

`DELETE /api/admin/documents/{source}` ([admin.py](v2/src/backend/routers/admin.py) L456-527) is store-agnostic: it always calls `search.delete_by_source(source)` (chunks, whichever provider) then, when a documents container is configured, `delete_document(source)` (blob, [files.py](v2/src/backend/services/files.py) L107-155). Returns `DeleteDocumentResponse{deleted:int, blob_deleted:bool}`. 404 only when **neither** a chunk **nor** a blob existed. Both stores implement `delete_by_source`: Azure Search pages client-side + deletes by id (BUG-0048); pgvector `DELETE ... WHERE title=$1 RETURNING id`.

**Orphan gap (Gap A):** the two deletes are **not transactional**. `delete_by_source` runs first, unguarded; if `delete_document` then throws a non-`ResourceNotFoundError` `AzureError`, chunks are already gone but the blob remains (endpoint 503s) — an **orphan blob** until a (safe, idempotent) retry. Also: pgvector's `url` column is declared but **never written**, so pgvector citations can't deep-link (FE falls back to `title` / `GET /api/files/{title}`).

### 4. Orchestrator × store — all four cells supported (ADR-0027 superseded ADR-0022)

| Orchestrator | Store | Retrieval | Citations |
|---|---|---|---|
| `langgraph` | pgvector | app-side embed → `PgVector.search(vector=)` dense cosine | shared `[docN]` seam |
| `langgraph` | AzureSearch | app-side embed → `AzureSearch.search(vector=)` hybrid + semantic re-rank | shared `[docN]` seam |
| `agent_framework` | AzureSearch | **server-side** Foundry IQ KB MCP tool | native `【N:M†source】` → `normalize_kb_citations` → `[docN]`+filename |
| `agent_framework` | pgvector | app-side embed → `PgVector.search(vector=)`, injected into the **user turn** | shared `[docN]` seam |

The store is **bound to the db type** (`DatabaseSettings._enforce_mode_consistency`), so there are two deployments; the orchestrator is the runtime-switchable axis (registry dispatch at [conversation.py](v2/src/backend/routers/conversation.py) ~L113; per-db-type default wired in `main.bicep` L1879 = `postgresql ? langgraph : agent_framework`, BUG-0064). **Stale-doc caveat:** `OrchestratorSettings.name` docstring + Hard Rule #20 R3 still say `agent_framework`×pgvector is 409-rejected — that guard was superseded by ADR-0027/BUG-0066 and the code now serves it; validation must assert the cell is **served**, not rejected.

## Technical Scenarios

### BUG-0088 fix — confirm-then-pin the Document Intelligence Office-format path

**Requirements:** `.docx` (and the other DI-routed Office/image formats) must parse → embed → index, on the live deployment, without dead-lettering.

**Preferred approach (evidence-driven, smallest change):**
1. **Reproduce + confirm first (implement/operator step).** Upload `MSFT_FY23Q4_10K.docx` to the current fresh `<AZD_ENV_NAME>` deployment and watch the pipeline (the same live method used to close BUG-0054: blob → `blob-events` → `blob_event`/`batch_push` → `cwyd-index`). Two outcomes:
   * **Ingests cleanly** → BUG-0088 was a stale-env/stale-image artifact of the retired deployment; **close it as fixed-by-redeploy** after validating the full file-type matrix. No code change.
   * **Still dead-letters** → read `az functionapp config appsettings list` for `AZURE_DOCUMENT_INTELLIGENCE_API_VERSION` / `_MODEL_ID`, and capture the `AzureError`. If the api-version is stale, unset the pin (fall back to the `2024-11-30` source default) or correct it; if `main.bicep` pins a stale value, fix the Bicep. Re-validate.
2. **Add a durable guard:** an opt-in live/integration test that parses a real `.docx` and asserts `> 0` chunks (the only test shape that can catch a service content-support regression — mocked-client unit tests cannot).
3. **Fix BUG-0055 (telemetry) in parallel or first** so the next such failure surfaces an exception instead of a blind dead-letter.

**Considered alternative (rejected as premature):** blindly bumping the api-version / re-deploying the function without capturing the live error — rejected because the source default is already correct, so a blind redeploy might "fix" it by luck without identifying whether a stale env pin or the image was at fault, and would not prevent recurrence.

### Full validation matrix (the hand-off to the implement phase)

A single end-to-end pass on the live deployment, reusing the BUG-0054 live-test method (Storage blob drop + `cwyd-index`/pgvector count + `GET /api/admin/documents`):

* **Ingestion — all 9 types:** upload one representative file per extension (`txt/md/json/html/pdf/docx/jpeg/jpg/png`); assert each yields `chunk_count > 0` and appears in `GET /api/admin/documents`. Note the UI only offers 3 — the other 6 must be dropped via Storage/Event-Grid or the admin API directly. **Decide (product):** widen the UI `ACCEPTED_EXTENSIONS` to the full 9 (Gap 1) or leave it 3-by-design.
* **Delete — chunks + blob:** per §3 checklist — for a representative doc, assert `DELETE` returns `{deleted:N, blob_deleted:true}`, then `GET /api/admin/documents` absent, store count 0, and `GET /api/files/{source}` → 404. Add the pgvector SQL count check on that profile.
* **pgvector integration:** on a `postgresql`-profile deployment (or local stack on a pgvector store), repeat ingestion + delete + a grounded chat.
* **Orchestrators × store:** validate each supported cell (switch via `PATCH /api/admin/config`, confirm via `/config/effective` + `/status`) with the canonical grounding questions (pgvector: Contoso remote-work, BUG-0065; AzureSearch: "employee benefits", BUG-0028); assert the ordered SSE frames (`reasoning` → `citation` `[docN]`+filename → `answer` → `conversation`) + the out-of-domain fallback negative. For `agent_framework`×AzureSearch also assert the `knowledge_base_retrieve` tool frame + no `【…†source】` leak.
