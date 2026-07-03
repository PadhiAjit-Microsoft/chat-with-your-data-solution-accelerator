# CWYD v2 — Upload / Ingestion File-Type & Parser Matrix (READ-ONLY research)

Status: Complete
Date: 2026-07-02
Scope: Which file types the v2 upload/ingestion pipeline supports, which parser handles each, the authoritative extension→parser routing, UI/backend/parser gaps, v1-vs-v2 parity, and recommended sample files for a validation pass.

All paths below are workspace-relative to `c:\workstation\Microsoft\github\cwyd-cdb`.

---

## Research questions

1. Complete table: extension | parser class | parse mechanism | registered? (file:line) | notes.
2. Authoritative extension→parser routing code (file:line) + unknown-extension fallback.
3. Mismatch between (a) UI allow-list, (b) backend accept-list, (c) parser capability — the silent-failure gaps.
4. v1-vs-v2 file-type parity (v1's 9 types vs v2 coverage).
5. Recommended representative sample files (one per parse mechanism) for a full upload-validation pass.

---

## 1. Complete extension → parser matrix

The closed set of supported extensions is the `ParserKey` StrEnum in
backend/core/providers/parsers/base.py L21-38 (9 members). Every parser self-registers
against the ingestion-only registry `Registry[type[BaseParser]]` created in
v2/src/functions/core/parsers/_instance.py L18.

| Extension | Parser class | Parse mechanism | Registered? (file:line) | Notes |
|-----------|-------------|-----------------|--------------------------|-------|
| `txt`  | `TextParser` | Plain UTF-8 decode (`errors="strict"`) → paragraph chunks split on blank lines | v2/src/functions/core/parsers/text_parser.py L41 (`@registry.register(ParserKey.TXT)`) | Pure-CPU; `requires_ai_services=False`. Malformed bytes raise `UnicodeDecodeError` → poison queue (intentional). |
| `md`   | `TextParser` | Same as txt (Markdown treated as UTF-8 text, paragraph chunking) | v2/src/functions/core/parsers/text_parser.py L42 | No Markdown-specific structure parsing; raw text indexed. |
| `json` | `TextParser` | Same as txt (JSON treated as UTF-8 text, paragraph chunking) | v2/src/functions/core/parsers/text_parser.py L43 | No JSON structure parsing; whole file split on blank lines. |
| `html` | `HtmlParser` | `bs4.BeautifulSoup(content, "html.parser")`, decompose `script`/`style`/`noscript`, `get_text(separator="\n")` → paragraph chunks | v2/src/functions/core/parsers/html_parser.py L40 (`@registry.register(ParserKey.HTML)`) | Pure-CPU (built-in `html.parser`, no lxml). Mirrors v1 BeautifulSoup `.get_text()` path. |
| `pdf`  | `DocumentIntelligenceParser` | Azure Document Intelligence `prebuilt-layout` (`begin_analyze_document`); one `Chunk` per page (`page.lines[*].content` joined by `\n`) | v2/src/functions/core/parsers/document_intelligence_parser.py L56 (`@registry.register(ParserKey.PDF)`) | Network parser; `requires_ai_services=True`. Needs `AZURE_AI_SERVICES_ENDPOINT`. |
| `docx` | `DocumentIntelligenceParser` | Azure DI; "pageless" → falls back to one `Chunk` per `result.paragraphs[*]` | v2/src/functions/core/parsers/document_intelligence_parser.py L55 (`@registry.register(ParserKey.DOCX)`) | Network parser; `requires_ai_services=True`. |
| `jpeg` | `DocumentIntelligenceParser` | Azure DI OCR; one `Chunk` per page (image OCR text) | v2/src/functions/core/parsers/document_intelligence_parser.py L57 (`@registry.register(ParserKey.JPEG)`) | Network parser; `requires_ai_services=True`. |
| `jpg`  | `DocumentIntelligenceParser` | Azure DI OCR (same as jpeg) | v2/src/functions/core/parsers/document_intelligence_parser.py L58 (`@registry.register(ParserKey.JPG)`) | Network parser; `requires_ai_services=True`. |
| `png`  | `DocumentIntelligenceParser` | Azure DI OCR (same as jpeg) | v2/src/functions/core/parsers/document_intelligence_parser.py L59 (`@registry.register(ParserKey.PNG)`) | Network parser; `requires_ai_services=True`. |

Registration is eager-fired at process start via side-effect imports in
v2/src/functions/core/parsers/registry.py L22-24 (`from . import text_parser / html_parser / document_intelligence_parser`).

`ParserKey` StrEnum members (the authoritative closed set):
backend/core/providers/parsers/base.py L30-38 — `TXT="txt"`, `MD="md"`, `JSON="json"`,
`HTML="html"`, `PDF="pdf"`, `DOCX="docx"`, `JPEG="jpeg"`, `JPG="jpg"`, `PNG="png"`.

`requires_ai_services` flag: declared on `BaseParser` (default `False`)
backend/core/providers/parsers/base.py L60-62; overridden `True` on
`DocumentIntelligenceParser` v2/src/functions/core/parsers/document_intelligence_parser.py L67.

---

## 2. Authoritative extension → parser routing + unknown-extension fallback

### Extension derivation (shared, one-way `backend.core` leaf)

- `parser_key_for_path(name)` — v2/src/backend/core/paths.py L11-18:
  `PurePosixPath(name).suffix.lstrip(".").lower()`. POSIX-style on every platform
  (blob paths + URL paths). This is the single extension-extraction seam.

### Runtime routing — the queue-trigger consumer (the authoritative dispatch)

- v2/src/functions/batch_push/blueprint.py:
  - `_parser_key_for_filename(filename)` L83-89 → delegates to `parser_key_for_path`.
  - `_execute(...)` L118-120: `parser_cls = ingestion_parsers_registry.registry.get(_parser_key_for_filename(message.filename))`.
  - This is where an uploaded blob's extension selects its parser at ingest time.

- `Registry.get(key)` — v2/src/backend/core/registry.py L64-74: normalizes to lowercase,
  returns the registered class, or **raises `KeyError`** with a message listing all
  available keys. There is **no default/fallback parser** — an unregistered extension
  is a hard `KeyError`.

### Unknown-extension behavior by ingestion path

| Path | Pre-validation? | Unknown-ext outcome |
|------|-----------------|---------------------|
| Admin file upload (`POST /api/admin/documents`) | YES — `validate_upload` (see §3) | Rejected **415** before the blob is stored. |
| Admin URL ingest (`POST /api/admin/documents/url`) | Partial — `_blob_name_for_url` rewrites unknown/ext-less URLs to `.html` (ingestion.py L73, L94-99) | Never reaches unknown ext; stored as `.html` → `HtmlParser`. |
| Functions `add_url` HTTP trigger (`POST /api/add_url`) | NO ext validation; ext-less → `txt` default | Ext present but unregistered → `registry.get` `KeyError` → `map_function_exceptions` → 5xx. |
| Reprocess-all (`POST /api/admin/documents/reprocess`) | **NO** per-blob validation — pure fan-out (admin.py L657+; `reprocess_all` ingestion.py delegates to `batch_start_handler`) | Blob with unregistered ext → enqueued → `batch_push` → `KeyError` → **poison queue** (silent to operator). |
| Event Grid `BlobCreated` (direct blob drop, `EVENT_GRID` trigger) | **NO** — blob bypasses the admin route entirely | Same as above → `batch_push` → `KeyError` → **poison queue**. |

`batch_push` failure handling: `@log_queue_errors("batch_push")` (blueprint.py L127) logs
and **re-raises** so the Functions runtime retries then poisons the message
(handler.py header notes "Any exception propagates so the Functions runtime applies its
retry / poison-queue policy").

### Divergent ext-less URL default (two different defaults)

- Functions `add_url` blueprint: `_DEFAULT_PARSER_KEY = "txt"`
  v2/src/functions/add_url/blueprint.py L74; `_parser_key_for_url` L86-96 falls back to it.
  Its own comment (L71-73) says "later phases that add an HTML parser will replace this
  default with the HTML key" — **this was never done**.
- Backend admin URL route: `_DEFAULT_URL_BLOB_EXT = "html"`
  v2/src/backend/services/ingestion.py L73; `_blob_name_for_url` L94-99 stores ext-less
  URLs as `.html` → `HtmlParser`.

Consequence: the SAME ext-less web page ingested via the Functions `POST /api/add_url`
trigger routes to `TextParser` (raw HTML markup indexed as text), but via the admin UI
(backend route) routes to `HtmlParser` (clean text). See Gap 3 in §3.

---

## 3. Mismatch: UI allow-list vs backend accept-list vs parser capability

### (a) UI allow-list — only 3 of 9

- `ACCEPTED_EXTENSIONS = [".pdf", ".docx", ".txt"]`
  v2/src/frontend/src/pages/admin/IngestData/IngestData.tsx L55.
- `validateFile(file)` L230-240: **hard client-side rejection** of any extension not in
  `ACCEPTED_EXTENSIONS`, returning an error string **before the wire** (not just a picker
  filter). L232-233 build the "Unsupported file extension" message.
- `accept={ACCEPTED_EXTENSIONS.join(",")}` on the `<input type="file">` L484 (picker filter).
- Hint text L459 and page docstring L10-11 both say "Accepts `.pdf,.docx,.txt`".

### (b) Backend accept-list — all 9 (authoritative)

- `validate_upload(filename, content_size, settings)` — v2/src/backend/services/ingestion.py L207-268:
  - 503 if no `documents_container` / `doc_processing_queue` (L219-226).
  - 422 if empty filename (L227-231).
  - **415** if `parser_key_for_path(filename) not in ingestion_parsers_registry.registry`
    (L232-243) — the registry is the authoritative supported set (Hard Rule #4). Detail
    includes `supported = sorted(registry.keys())` (all 9).
  - 503 if resolved parser `requires_ai_services` and `AZURE_AI_SERVICES_ENDPOINT` is not
    an `https://` URL (L244-259) — blocks DI-routed types (pdf/docx/jpeg/jpg/png) when DI
    is unconfigured, rather than poisoning the queue.
  - 413 if `content_size > MAX_UPLOAD_SIZE_BYTES` (50 MiB, L60) (L260-268).
- Route wiring: `upload_document_endpoint` v2/src/backend/routers/admin.py L607-646 calls
  `validate_upload(...)` at L636 and maps `UploadRejected` → `HTTPException` (L637-640).

### (c) Parser capability — all 9 (see §1)

### The gaps (where uploads silently or surprisingly fail)

- **Gap 1 (biggest — UI regression, hard reject).** UI offers only `.pdf/.docx/.txt`.
  The 6 types `md, json, html, jpeg, jpg, png` are fully supported by the backend + parsers
  but **cannot be uploaded through the admin UI file-upload** — `validateFile` rejects them
  client-side (IngestData.tsx L232-233). They are only reachable via: admin URL ingest
  (html/pdf-by-URL), reprocess-all, direct blob drop (Event Grid), or the Functions HTTP
  triggers. This is a **visible** rejection (error message), not a silent one, but it is a
  parity regression vs v1 (§4).

- **Gap 2 (silent failure — non-UI paths).** The admin upload route blocks unknown
  extensions (415), but **reprocess-all and Event-Grid blob-drop do NOT pre-validate the
  extension**. A blob with an unsupported extension (e.g. `.xlsx`, `.csv`, `.zip`) already
  in / dropped into the documents container → `batch_push` → `registry.get(unknown)` →
  `KeyError` → retries → **poison queue**. No UI signal; only App Insights telemetry
  (`log_queue_errors`). This is the true "uploads silently fail" surface.

- **Gap 3 (silent content pollution — URL entry-point divergence).** Ext-less URL via the
  Functions `POST /api/add_url` trigger defaults to `TextParser` (indexes raw HTML markup),
  while via the admin UI (backend route) it defaults to `HtmlParser` (clean text). Same URL,
  different quality depending on entry point. Root: stale `_DEFAULT_PARSER_KEY = "txt"` in
  add_url/blueprint.py L74 (its own comment says it should have become `html`).

- **Gap 4 (unrealized DI coverage).** `DocumentIntelligenceParser`'s docstring claims the
  `prebuilt-layout` model natively handles XLSX, PPTX, and HTML too
  (document_intelligence_parser.py L26-34), but those extensions are **not** `ParserKey`
  members and **not** registered, so `.xlsx` / `.pptx` uploads → 415. Adding `ParserKey`
  members + `@registry.register(...)` decorators would extend coverage with no new SDK call.

- **Gap 5 (DI-config gate consistency).** DI-routed types require `AZURE_AI_SERVICES_ENDPOINT`.
  Admin upload blocks them at the boundary (503) when unset (validate_upload L244-259). But
  via reprocess/Event-Grid, a `.pdf` blob → `batch_push` → `DocumentIntelligenceParser._get_client`
  raises `ValueError` (endpoint not https, document_intelligence_parser.py L96-103) → poison
  queue. Same non-UI silent-failure class as Gap 2.

---

## 4. v1 vs v2 file-type parity

### v1's 9 types (authoritative source)

docs/supported_file_types.md L5-15: **PDF, JPEG, JPG, PNG, TXT, HTML, MD (Markdown), DOCX, JSON.**

### v2 coverage

- **Parser / backend level: 9/9 — FULL PARITY.** All 9 are `ParserKey` members and registered
  (§1). Guarded against regression by
  v2/tests/functions/core/parsers/test_supported_extensions.py (added in BUG-0074).
- **UI file-upload level: 3/9 — REGRESSION.** v2 UI hardcodes `.pdf/.docx/.txt`
  (IngestData.tsx L55). v1's UI offered **all configured document processors dynamically**:
  `type=file_type` where `file_type = [p.document_type for p in config.document_processors]`
  (code/backend/pages/01_Ingest_Data.py L104-113). So v1 exposed the full configured set in
  the picker; v2 exposes only 3.

### History / context

BUG-0074 (v2/docs/bugs.md L1272-1305, fixed 2026-06-22) is where v2 reached parser-level
parity: it added the bs4 `HtmlParser`, registered `md`/`json` on `TextParser` and
`jpeg`/`jpg`/`png` on the DI parser (v2 previously registered only pdf/docx/txt), and set
`_blob_name_for_url` to default web pages to `.html`. The **UI `ACCEPTED_EXTENSIONS` was not
widened** in that fix, so the 3/9 UI gap remains.

Net parity status:
- Full-pipeline parity (via URL ingest / reprocess / direct blob drop / Functions triggers): **9/9**.
- Admin-UI file-upload parity: **3/9** (missing md, json, html, jpeg, jpg, png).

---

## 5. Recommended representative sample files for a full upload-validation pass

One per parse mechanism (covers all 3 mechanisms × all 9 extensions), plus negative cases
that exercise the routing gaps.

### Positive — one per registered extension (validates each registration + mechanism)

| # | Sample file | Extension | Exercises | Expected chunks |
|---|-------------|-----------|-----------|-----------------|
| 1 | `sample.txt`  | txt  | `TextParser` plain-text mechanism | 1+ paragraph chunks |
| 2 | `sample.md`   | md   | `TextParser` on Markdown | 1+ paragraph chunks (raw text) |
| 3 | `sample.json` | json | `TextParser` on JSON | 1+ paragraph chunks (raw text) |
| 4 | `sample.html` | html | `HtmlParser` bs4 script/style stripping | clean-text paragraph chunks (no tags) |
| 5 | `sample.pdf`  | pdf  | `DocumentIntelligenceParser` page-chunk path | one chunk per page |
| 6 | `sample.docx` | docx | DI paragraph-fallback path (pageless) | one chunk per paragraph |
| 7 | `sample.jpeg` | jpeg | DI OCR | OCR text chunk(s) |
| 8 | `sample.jpg`  | jpg  | DI OCR (verify jpg == jpeg registration) | OCR text chunk(s) |
| 9 | `sample.png`  | png  | DI OCR | OCR text chunk(s) |

Mechanism coverage minimum (if only 3 files): pick one TextParser file (`.txt`), the
`.html` file, and one DI file (`.pdf`).

### Negative / edge — validate the gaps in §3

| # | Sample / action | Validates |
|---|-----------------|-----------|
| N1 | `sample.xlsx` (unregistered ext) via admin upload | **415** at `validate_upload` (Gap 4). |
| N2 | `sample.xlsx` dropped directly into documents container (Event Grid) or present during reprocess-all | `batch_push` `KeyError` → poison queue (**Gap 2** — silent). |
| N3 | File with no extension (e.g. `README`) via admin upload | **415** (`parser_key_for_path` → `""` not in registry). |
| N4 | Ext-less URL (e.g. `https://example.com/article`) via **admin UI** vs via **Functions `/api/add_url`** | admin → `.html`/`HtmlParser` (clean); Functions → `txt`/`TextParser` (raw markup) — **Gap 3**. |
| N5 | Image-only / blank `.pdf` (no extractable text) | zero-chunk warning path (handler.py L94-105) — indexed-but-empty, no poison. |
| N6 | Any DI type (`.pdf`) with `AZURE_AI_SERVICES_ENDPOINT` unset | admin upload → **503** (validate_upload L244-259); reprocess/Event-Grid → poison (**Gap 5**). |
| N7 | `>50 MiB` file via admin upload | **413** (validate_upload L260-268); UI also blocks at `validateFile` L235-238. |
| N8 | Any of md/json/html/jpeg/jpg/png via admin **UI** file-upload | UI hard-reject (**Gap 1**, IngestData.tsx L232-233) — confirms these 6 are UI-unreachable. |

Note: check the repo `data/` folder (`data/contract_data/`, `data/sample_code/`) for
existing sample documents before authoring new ones.

---

## Key files (evidence index)

- Parsers: v2/src/functions/core/parsers/text_parser.py, html_parser.py,
  document_intelligence_parser.py, registry.py, _instance.py.
- ParserKey + BaseParser + `requires_ai_services`: v2/src/backend/core/providers/parsers/base.py.
- Registry primitive (`get` raises KeyError, no fallback): v2/src/backend/core/registry.py.
- Extension extraction: v2/src/backend/core/paths.py.
- Runtime routing (queue trigger): v2/src/functions/batch_push/blueprint.py, handler.py.
- URL routing (Functions): v2/src/functions/add_url/blueprint.py.
- Upload-side validation + URL blob naming: v2/src/backend/services/ingestion.py.
- Download / delete (filename validation): v2/src/backend/services/files.py.
- Admin routes (upload/url/reprocess/delete): v2/src/backend/routers/admin.py.
- Frontend upload UI + accept gate: v2/src/frontend/src/pages/admin/IngestData/IngestData.tsx.
- v1 supported types: docs/supported_file_types.md.
- v1 UI dynamic accept list: code/backend/pages/01_Ingest_Data.py L104-113.
- Parity history: v2/docs/bugs.md L1272-1305 (BUG-0074).
- Regression guard: v2/tests/functions/core/parsers/test_supported_extensions.py.

---

## Clarifying questions (none blocking)

- None required to answer the research questions. If a follow-up wants a fix, the highest-value
  targets are: widen `ACCEPTED_EXTENSIONS` to the full registry key set (Gap 1), pre-validate
  extension in the reprocess/Event-Grid path (Gap 2), and align the `add_url` blueprint default
  to `html` (Gap 3).

## Follow-on research not completed (out of scope for these questions)

- [ ] Confirm whether the Event Grid `blob_event` trigger has any extension filter at the
      storage-subscription level (Bicep) that would block unregistered extensions before they
      reach `batch_push` (would narrow Gap 2). Not read in this session.
- [ ] Confirm the `search_skill` / integrated-vectorization path (if any) applies its own
      file-type handling distinct from `batch_push`.
