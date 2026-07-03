<!-- markdownlint-disable-file -->
# Research: CWYD v2 document-DELETE contract + pgvector integration points

Status: Complete
Date: 2026-07-02
Scope: READ-ONLY mapping. Root: c:\workstation\Microsoft\github\cwyd-cdb\v2

## Research questions

1. Exact DELETE flow (file:line): endpoint → delete_by_source (chunks) → delete_document (blob) → response. Does it delete BOTH for both stores? Failure / partial-delete semantics?
2. pgvector provider surface: schema (columns, indexes), insert + delete + search, dense-vs-FTS retrieval, known fragilities.
3. Validation checklist for "delete a file → chunks gone AND blob gone", separately for AzureSearch and pgvector.
4. Gaps where delete could leave orphans (blob deleted but chunks remain, or vice versa; URL-sourced docs).

---

## 1. The exact DELETE flow

### Endpoint: `DELETE /api/admin/documents/{source:path}`

File: v2/src/backend/routers/admin.py — `delete_document_endpoint`

- Route decorator + signature: L456-467 (`@router.delete("/documents/{source:path}", response_model=DeleteDocumentResponse, status_code=200)`).
- `{source:path}` converter (L457) captures URL-typed sources containing slashes; FastAPI percent-decodes before the handler runs (docstring L470-473).
- Dependencies injected: `settings: SettingsDep`, `credential: CredentialDep`, `search: SearchProviderDep`, `_user: AdminUserIdDep` (L462-466).

Ordered body (L501-533):

1. L501-505 — guard: if `search is None` → raise `503` ("Search backend is not configured for this deployment."). Route stays mounted so operators discover the gap explicitly rather than routing-404-ing.
2. L502 — `deleted = await search.delete_by_source(source)` — deletes index/store chunks. Returns the count of chunks removed.
3. L503 — `blob_deleted = False` (default).
4. L504-512 — blob deletion, attempted **only** when `settings.storage.documents_container` is set:
   - L506-508 — `blob_deleted = await delete_document(source, settings=settings, credential=credential)`.
   - L509-512 — `except ValueError: blob_deleted = False` — URL-typed sources carry path separators, fail `_validate_filename`, and have no backing blob; swallowed to `False`.
5. L513-517 — 404 logic: `if deleted == 0 and not blob_deleted:` → raise `404` ("No indexed chunks or source blob found for source {source!r}."). 404 fires **only when NEITHER a chunk NOR a blob existed**.
6. L518-526 — `logger.info("Admin deleted document.", extra={operation, source, deleted_count, blob_deleted})`.
7. L527 — `return DeleteDocumentResponse(deleted=deleted, blob_deleted=blob_deleted)`.

### Response model

File: v2/src/backend/models/admin.py — `DeleteDocumentResponse` L123-137.

- `deleted: int` (L126-130), `ge=0`, "Number of indexed chunks removed for the source."
- `blob_deleted: bool` (L131-136), `default=False`, "Whether the source blob was removed … (False for URL-typed sources, which have no blob)." Default keeps the FE OpenAPI client non-breaking (added by BUG-0073).

### Chunk deletion — per store

Provider is resolved at request time from `app.state.search_provider` (see §4 selection). Both stores implement `BaseSearch.delete_by_source(source) -> int` (contract: v2/src/backend/core/providers/search/base.py L109-124).

- Azure AI Search: v2/src/backend/core/providers/search/azure_search.py `delete_by_source` L209-256.
  - `title` is **searchable but NOT filterable** in the deployed index, so a server-side `$filter` is rejected (BUG-0048). Method pages every chunk client-side: `client.search(search_text="*", select=["id","title"])` (L224-229), keeps `id`s whose `title == source` by plain Python equality (L230-234), deletes in ≤1000-id batches via `client.delete_documents(documents=[{"id": ...}])` (L235-243). Returns `deleted_count` (L256).
  - Wraps `AzureError` → `logger.exception(...)` + re-raise (L244-255), Hard Rule #14.
- pgvector: v2/src/backend/core/providers/search/pgvector.py `delete_by_source` L167-188.
  - `sql = f"DELETE FROM {self._table} WHERE title = $1 RETURNING id"` (L171), `await self._pool.fetch(sql, source)` (L174-177), returns `len(rows)` (L188).
  - Same `title` field as Azure Search (ingestion writes filename/URL there). Wraps `asyncpg.PostgresError` → log + re-raise (L178-187).

### Blob deletion

File: v2/src/backend/services/files.py — `delete_document` L107-155 (imported at admin.py L96).

- Signature L107-112: `delete_document(filename, *, settings, credential) -> bool`.
- L132 — `_validate_filename(filename)` runs before any SDK call.
- L133-139 — builds a `ContainerClient(account_url=blob_endpoint, container_name=documents_container, credential=…)` and `await container_client.delete_blob(filename)` → `return True` (L140).
- L141-143 — `except ResourceNotFoundError: return False` — already-absent blob is an idempotent no-op success, NOT an error.
- L144-154 — `except AzureError: logger.exception(...) + raise` (Hard Rule #14).
- `_validate_filename` (L50-72): rejects empty (L64), >255 chars (L66), `/` or `\` (L68), `..` (L70), control chars (L71-72) → `ValueError`. This is what rejects URL-typed sources (they contain `/`).

### Does it delete BOTH for both stores?

Yes — the endpoint is store-agnostic: it always calls `search.delete_by_source(source)` (chunks, whichever provider is wired) AND, when a documents container is configured, `delete_document(source)` (blob). The two-part removal is by design so a deleted document becomes fully unreachable (docstring L475-485; introduced by BUG-0073, which previously deleted only the chunks and left the blob downloadable via `GET /api/files/{filename}`).

### Failure / partial-delete semantics

- **No transaction / no rollback across the two deletes.** `delete_by_source` runs FIRST (L502) and is NOT wrapped in try/except at the endpoint — if it raises `AzureError` / `asyncpg.PostgresError`, it propagates to the app-level handler (sanitized to 503) and the blob delete is never attempted. So a chunk-delete failure leaves the blob intact (no orphan created, but the delete is incomplete and must be retried).
- If `delete_by_source` succeeds (chunks gone) but `delete_document` then raises a non-`ResourceNotFoundError` `AzureError`, that exception propagates (only `ValueError` is caught at L509). Result: chunks already deleted, blob NOT deleted, endpoint returns 503 → **orphan blob** until retried. Retry is safe/idempotent (chunk delete returns 0 next time, blob delete completes).
- `ValueError` from `delete_document` (URL-typed source) is the ONLY blob-delete exception swallowed (L509-512) → `blob_deleted=False`, and the response can still be 200 if chunks were removed.
- 404 is returned only when `deleted == 0 AND not blob_deleted` (L513) — i.e. nothing existed to delete in either store.
- Idempotency: re-deleting an already-deleted source returns 404 (0 chunks, no blob). Missing blob alone → `False`, not an error.

---

## 2. pgvector provider surface

File: v2/src/backend/core/providers/search/pgvector.py (registered `@registry.register("pgvector")` L69; class `PgVector(BaseSearch)` L70).

### Schema (`ensure_schema`, L285-339)

Single-flight bootstrap: `self._schema_ready` flag + `self._schema_init_lock` (L88-91), double-checked inside the lock (L290-297). DDL (L303-323, executed via `self._pool.execute(sql)` L324):

```
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE IF NOT EXISTS documents (
    id              TEXT PRIMARY KEY,
    content         TEXT NOT NULL,
    title           TEXT,
    url             TEXT,
    last_modified   TIMESTAMPTZ NOT NULL DEFAULT now(),
    content_vector  vector(<dims>) NOT NULL
);
ALTER TABLE documents ADD COLUMN IF NOT EXISTS
    last_modified TIMESTAMPTZ NOT NULL DEFAULT now();
CREATE INDEX IF NOT EXISTS documents_vec_hnsw
    ON documents USING hnsw (content_vector vector_cosine_ops);
```

- Columns: `id` TEXT PK, `content` TEXT NOT NULL, `title` TEXT (the source filename/URL — the delete/list key), `url` TEXT, `last_modified` TIMESTAMPTZ NOT NULL DEFAULT now(), `content_vector` vector(dims) NOT NULL.
- `dims` = `settings.openai.embedding_dimensions` (L298). Changing dims on an existing deploy needs a manual drop+recreate — `CREATE TABLE IF NOT EXISTS` won't alter an existing column (docstring L26-28).
- The explicit `ALTER TABLE … ADD COLUMN IF NOT EXISTS last_modified` (L318-319) is the BUG-0075 idempotent migration for deploys whose table predates the column; existing rows take `now()` at migration time until re-ingested (comment L314-317).
- Index: `documents_vec_hnsw` HNSW over `content_vector vector_cosine_ops` (L320-322). No index on `title` — `delete_by_source` / `list_sources` do a full-table `WHERE title=` / `GROUP BY title` scan.
- `PostgresError` on DDL → log + re-raise (L326-337).

### Construction / DI (L71-92)

- `__init__(settings, credential, *, pool: asyncpg.Pool, table: str = "documents")`. Pool is REQUIRED — no lazy-construct fallback (comment L80-86). Shares the single per-process pool with the chat-history `PostgresClient` (lifespan calls `ensure_pool()` then hands the pool here).
- `aclose()` L279-282 is a deliberate no-op: pool ownership stays with `PostgresClient` — closing here would kill chat-history too.
- `_table` is allow-listed at construction (never user-supplied) so it is safe to interpolate into SQL; all VALUES are parameterized (comments L110-112, L303-306).

### Insert / upsert (`merge_or_upload_documents`, L233-277)

- Empty batch → `return []` (L237-238).
- Builds one multi-row `INSERT … VALUES (…)` with 4 positional params/row: `(id, content, title, content_vector::vector)` (L246-263). `$N::vector` casts the text literal from `_format_vector_literal` (asyncpg has no native vector codec).
- `ON CONFLICT (id) DO UPDATE SET content=EXCLUDED.content, title=EXCLUDED.title, content_vector=EXCLUDED.content_vector, last_modified=now() RETURNING id` (L264-271). Re-ingesting the same chunk id refreshes `last_modified`.
- **`url` is NEVER written** — the INSERT column list is `(id, content, title, content_vector)` only (L262); the ON CONFLICT clause does not touch `url` either. `url` stays NULL. (Root cause: both ingestion paths build `SearchDocument` without `url=` — batch_push handler v2/src/functions/batch_push/handler.py `_build_document` L57-71 ("`url` is intentionally omitted") and add_url handler v2/src/functions/add_url/handler.py L101-105.)
- `PostgresError` → log + re-raise (L272-276). Returns raw asyncpg `Record` list typed `list[Any]` (Hard Rule #11(a) boundary).

### Retrieval (`search`, L94-166)

Two mutually exclusive modes selected by whether a `vector` is passed:

- **Dense (vector provided)** L127-139:
  `SELECT id, content, title, url, 1 - (content_vector <=> $1::vector) AS score FROM documents [WHERE <filter>] ORDER BY content_vector <=> $1::vector LIMIT $2`. `<=>` is pgvector cosine distance; score = `1 - distance` (cosine similarity). Optional `filter_expression` interpolated as a raw `WHERE` fragment (L135-136). This is the path BUG-0065 required (`langgraph` must pass `vector=`).
- **FTS fallback (no vector)** L140-153:
  `SELECT id, content, title, url, ts_rank(to_tsvector('english', content), plainto_tsquery('english', $1)) AS score FROM documents WHERE to_tsvector('english', content) @@ plainto_tsquery('english', $1) [AND (<filter>)] ORDER BY score DESC LIMIT $2`. `$1` = the free-text `query`.
- `top_k` = `top_k` arg or `settings.search.top_k` (L109-110). `use_semantic_search` is accepted for BaseSearch-seam parity and intentionally ignored — pgvector has no re-ranking mode (L107-108, L114-118).
- Maps each row → `SearchResult(id, content, title, url, score)` (L157-166); `url` is coalesced to `""` (L162) — and is always empty in practice because the column is never populated.
- `PostgresError` → log + re-raise (L149-156).

### `list_sources` (L189-232) and `get_document_by_key`

- `list_sources`: `SELECT title AS source, COUNT(*) AS chunk_count, MAX(last_modified) AS last_modified FROM documents WHERE title IS NOT NULL GROUP BY title ORDER BY title` (L195-201). Emits `last_modified` as ISO-8601 or `None` (L225-229). This is what the admin Delete Data grid lists; `source` round-trips into `delete_by_source`.
- `get_document_by_key`: base ABC default raises `NotImplementedError` (base.py L146-165); whether pgvector overrides it was not read this turn (out of the DELETE scope).

### Known fragilities (pgvector)

- **`url` column is dead weight** — declared + selected but never written, so `SearchResult.url` from pgvector is always `""`. Citations grounded on pgvector cannot deep-link to an external URL; the FE must fall back to `title`/`GET /api/files/{title}`.
- **No index on `title`** — `delete_by_source`, `list_sources` do full scans. Fine at CWYD document volumes; a fragility only at very large corpora.
- **Embedding-dimension drift** — `content_vector vector(<dims>)` is fixed at first bootstrap; changing the embedding model's dims requires manual drop+recreate (docstring L26-28).
- **HNSW-only** — no exact/IVFFlat option; approximate recall is acceptable but non-tunable through this provider.
- **FTS is `'english'`-hardcoded** (L131, L134, L146-147) — non-English content ranks poorly on the text-fallback path.
- **`aclose()` is a no-op by design** — provider must never own the pool lifecycle.

---

## 3. Validation checklist — "delete a file → chunks gone AND blob gone"

Preconditions common to both: know the exact `source` value (the `title` on every chunk = the blob filename for file/backend-URL ingests). Auth as admin (local dev falls back to `local-dev` when Easy Auth headers absent).

### Which store is live

`GET /api/admin/status` surfaces the active `index_store` (see §4). In this deployment it is pgvector (BUG-0073 note: "the index **is** pgvector (one store)").

### AzureSearch

Before delete:

1. `GET /api/admin/documents` → confirm the `source` appears with `chunk_count = N > 0`.
2. Chunk count direct: Search index query `search=*` counting docs where `title == source` (or the portal "Search explorer"). Index has `title` non-filterable — must page + match client-side, mirroring the provider.
3. Blob exists: `GET /api/files/{source}` → 200 (streams bytes), OR list the documents container blob `{source}`.

Delete: `DELETE /api/admin/documents/{source}` → expect `200 {"deleted": N, "blob_deleted": true}` (or `blob_deleted:false` if no container / URL-typed).

After delete:

4. `GET /api/admin/documents` → `source` absent (or `chunk_count` dropped by N).
5. Index doc count for `title == source` → 0.
6. `GET /api/files/{source}` → 404 (blob gone).
7. Re-`DELETE` the same source → `404` (idempotent: 0 chunks, no blob).

### pgvector

Before delete (SQL on the `documents` table via the pg tooling):

1. `GET /api/admin/documents` → `source` present, `chunk_count = N`.
2. `SELECT COUNT(*) FROM documents WHERE title = '<source>';` → N.
3. Blob exists: `GET /api/files/{source}` → 200, OR container blob `{source}` present.

Delete: `DELETE /api/admin/documents/{source}` → `200 {"deleted": N, "blob_deleted": true}`.

After delete:

4. `SELECT COUNT(*) FROM documents WHERE title = '<source>';` → 0.
5. `GET /api/admin/documents` → `source` absent.
6. `GET /api/files/{source}` → 404.
7. Re-`DELETE` → `404`.

Cross-store note: `deleted` in the response is authoritative for chunk removal count; `blob_deleted` is authoritative for the blob. A green run is `deleted == N (pre-count) AND blob_deleted == true AND GET /api/files/{source} == 404 AND row/doc count == 0`.

---

## 4. Orphan gaps

### Provider selection (`index_store`)

- `get_search_provider` (v2/src/backend/dependencies.py L111-124) simply returns `getattr(request.app.state, "search_provider", None)`. The concrete provider (azure_search vs pgvector) is chosen at **lifespan startup** keyed on `settings.search.index_store` and stashed on `app.state.search_provider`; the request layer is store-agnostic. `SearchProviderDep` is `BaseSearch | None` — `None` when no search endpoint is configured → every admin doc route 503s the search path (mounted, not routing-404).
- `index_store` is a registry-driven `IndexStore | str` settings field (Hard Rule #4/#11 carve-out); the registry lives in `backend.core.providers.search.registry`.

### Gap A — chunk-delete succeeds, blob-delete throws (non-404 AzureError) → orphan blob

`delete_by_source` runs first and un-guarded (admin.py L502); `delete_document` is only guarded against `ValueError` (L509). If `delete_blob` raises any other `AzureError` (throttling, transient 5xx, auth), the chunks are already gone but the blob is not; the endpoint returns 503. The blob is orphaned until the delete is retried. Retry is idempotent (chunk delete → 0, blob delete completes), but nothing auto-retries.

### Gap B — chunk-delete throws → blob never attempted → incomplete delete

If `delete_by_source` raises (`AzureError` / `asyncpg.PostgresError`), it propagates before `delete_document` is reached (L502). Chunks may be partially deleted (Azure Search deletes in ≤1000-id batches L235-243; a mid-loop failure leaves earlier batches deleted), and the blob is untouched. No rollback. Response is 503.

### Gap C — URL-sourced docs: the two URL ingest paths diverge on `title`

This is the sharpest orphan risk.

- **Backend admin URL ingest (BUG-0074 path)** — `services/ingestion.py` `ingest_url` (L112-160) derives a **flat, separator-free** blob name via `_blob_name_for_url` (L81-110, e.g. `host_path.html`), uploads it, and lets it flow through `batch_push`. In `batch_push`, `parser.parse(content, source=message.filename)` (handler L89) sets `chunk.source = <flat blob name>`, so `title = <flat blob name>`. Delete works end-to-end: `delete_by_source(<flat name>)` clears chunks AND `delete_document(<flat name>)` passes `_validate_filename` (no slashes) → blob removed. **No orphan.**
- **Functions `add_url` blueprint (standalone HTTP trigger)** — `add_url/handler.py` `parser.parse(content, source=request.url)` (L131) → `chunk.source = request.url` → `SearchDocument.title = request.url` (the **full URL, with `://` and `/`**) (L101-105). If a doc is ingested via this path, its chunks are indexed under the full URL. On delete:
  - `delete_by_source(<full URL>)` → removes chunks ✓.
  - `delete_document(<full URL>)` → `_validate_filename` rejects (contains `/`) → `ValueError` → swallowed → `blob_deleted=False` (admin.py L509-512). Any blob that path may have written is **orphaned** (never deletable through the admin route), and the endpoint can still 200 on the chunk removal, masking the orphan.
- Net: whether a URL-sourced document is fully deletable depends on WHICH ingest path created it. The backend admin path is safe; the Functions `add_url` blueprint path leaves a delete-blind blob whenever it stores one. Worth confirming whether the Functions `add_url` blueprint writes a blob at all, or only indexes — if it only indexes (no blob), there is no orphan blob, only the (harmless) `blob_deleted=False`.

### Gap D — reverse direction (blob deleted out-of-band → chunks remain)

Not handled by this endpoint. Deleting a blob directly in Storage (not via the admin route) leaves the index chunks intact and still retrievable in chat. BUG-0077 (the inverse — auto-de-index on `BlobDeleted` via a `blob_event` translator mapping `subject` → `delete_by_source`) is the tracked counterpart; referenced by BUG-0073's cross-links. Confirm BUG-0077 status separately (out of this turn's read scope).

### Gap E — multi-store deployments

The endpoint deletes from exactly ONE search provider (`app.state.search_provider`). If a deployment ever indexed the same corpus into both azure_search and pgvector, delete only touches the configured one. Current deployment is single-store (pgvector), so not live — but the contract does not fan out.

---

## Evidence index (file:line)

- Endpoint: v2/src/backend/routers/admin.py L456-527 (decorator L456-460, signature L461-467, body L501-527, 404 logic L513-517, response L527).
- Response model: v2/src/backend/models/admin.py L123-137.
- Blob delete: v2/src/backend/services/files.py `delete_document` L107-155; `_validate_filename` L50-72; import at admin.py L96.
- Azure Search delete: v2/src/backend/core/providers/search/azure_search.py `delete_by_source` L209-256; `list_sources` L258-296.
- pgvector: v2/src/backend/core/providers/search/pgvector.py — `search` L94-166, `delete_by_source` L167-188, `list_sources` L189-232, `merge_or_upload_documents` L233-277, `aclose` L279-282, `ensure_schema` L285-339; schema docstring L11-21; register L69.
- BaseSearch contract: v2/src/backend/core/providers/search/base.py — `delete_by_source` L109-124, `search` L74-107, `list_sources` L126-150, `SourceListing` L39-63.
- Provider selection: v2/src/backend/dependencies.py `get_search_provider` L111-124.
- Ingestion source→title: v2/src/functions/batch_push/handler.py `_build_document` L57-71 (title=chunk.source, url omitted), `parse(source=message.filename)` L89; v2/src/functions/add_url/handler.py L101-105 (title=chunk.source), `parse(source=request.url)` L131; v2/src/backend/services/ingestion.py `_blob_name_for_url` L81-110, `ingest_url` L112-160.
- SearchDocument/SearchResult types: v2/src/backend/core/types.py `Chunk.source` L150, `Citation` L171-181, `SearchResult` L187-198, `SearchDocument` L203+.
- Bugs: v2/docs/bugs.md — BUG-0048 L884+, BUG-0073 L1256-1270, BUG-0074 L1272-1288, BUG-0075 L1290+, BUG-0065/0066 L1156-1166.

## Follow-on (in scope, not resolved this turn)

- [ ] Confirm whether the Functions `add_url` blueprint writes a blob (Gap C severity hinges on it).
- [ ] Confirm BUG-0077 status (auto-de-index on `BlobDeleted`) — the Gap D counterpart.
- [ ] Confirm pgvector overrides `get_document_by_key` (used by agent_framework citation enrichment; not read this turn).

## Clarifying questions

None — the DELETE contract and pgvector surface are fully mapped from source.
