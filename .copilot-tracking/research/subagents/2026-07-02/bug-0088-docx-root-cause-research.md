# BUG-0088 — `.docx` ingestion dead-letters (0 chunks) — Root-cause research

Date: 2026-07-02
Scope: READ-ONLY research. No code changed.
Status: **Complete** (root cause narrowed to a single mechanism + top hypothesis; definitive confirmation needs one live-endpoint parse capture, which is an operator/runtime action, not a source read).

## Research questions

1. Exact current-source `.docx` code path (file:line) from queue message → chunks.
2. Does current source correctly handle `.docx` (paragraph fallback present)? Quote it.
3. Root-cause hypothesis: stale-deploy (BUG-0049 fix not deployed) vs new bug — with evidence.
4. What throws for `.docx` but not PDF (dead-letter ⟹ throw, not silent-0). Reconcile.
5. Recommended fix + how to definitively repro/validate (which test file).

---

## Executive summary (answer first)

- **BUG-0088 is NOT a stale-deploy manifestation of BUG-0049.** BUG-0049 was a *silent* 0-chunk failure that returns `200`/success and **never dead-letters**. BUG-0088 **dead-letters after 5 retries**, which can only happen when an exception is **raised** and re-raised. The current handler treats "0 chunks" as a logged warning + `return []` (no raise), and the *pre*-BUG-0049 handler also returned `[]` on 0 chunks — so **neither** a current nor a stale parser image dead-letters on the empty-chunk path. Dead-letter therefore proves a **throw**, which is a *different* failure class than BUG-0049.
- The current source **does** contain the BUG-0049 paragraph fallback (and it is unit-tested). So the parser code is not the regression.
- `.docx` and PDF route to the **same** `DocumentIntelligenceParser`, same `model_id`, same `api_version`, same embedder, same search push. The **only** divergence between the two is the byte content-type handed to Document Intelligence. So the throw is almost certainly **Document Intelligence rejecting the Office/DOCX content** inside `begin_analyze_document` → `AzureError` → re-raised → runtime retries 5× → poison.
- **Top hypothesis (config defect, NEW bug):** the *deployed* runtime's Document Intelligence `api_version` (or `model_id`) does not support Office formats. Office-format support (DOCX/XLSX/PPTX/HTML) for `prebuilt-layout` requires api-version `2024-02-29-preview` or later (GA `2024-11-30`). The **source default is correct** (`2024-11-30` / `prebuilt-layout`), but both fields are **operator-pinnable via env vars**, so a stale/older env pin on the Function App would make `.docx` throw while PDF (supported at every api-version) succeeds.
- **Unit tests cannot catch this** — every Document Intelligence test uses a **mocked** client, so the real service's Office-format handling is never exercised. That is why 20+ DI parser tests are green while cloud `.docx` fails.

---

## Q1 — Exact current-source `.docx` code path (queue message → chunks)

Queue message → chunks, in order:

1. Queue trigger fires.
   - v2/src/functions/batch_push/blueprint.py — `@bp.queue_trigger(arg_name="msg", queue_name="%AZURE_DOC_PROCESSING_QUEUE%", connection="AzureWebJobsStorage")` decorating `async def batch_push(msg)` (approx L155–L169). Wrapped by `@log_queue_errors("batch_push")`.
   - Body: `message = parse_push_message(msg)` then `await _execute(message, get_settings())`.
2. Collaborator resolution + parser routing.
   - v2/src/functions/batch_push/blueprint.py — `_execute(...)` (approx L93–L145). Parser is resolved by extension:
     - `parser_cls = ingestion_parsers_registry.registry.get(_parser_key_for_filename(message.filename))` (approx L121–L123).
   - v2/src/functions/batch_push/blueprint.py — `_parser_key_for_filename` (approx L74–L82) delegates to `parser_key_for_path`.
   - v2/src/backend/core/paths.py — `parser_key_for_path` (L11–L19): `PurePosixPath(name).suffix.lstrip(".").lower()` → `MSFT_FY23Q4_10K.docx` → `"docx"`.
3. `"docx"` → `DocumentIntelligenceParser`.
   - v2/src/functions/core/parsers/document_intelligence_parser.py — `@registry.register(ParserKey.DOCX)` stacked on the class (approx L59–L64). `ParserKey.DOCX = "docx"` in v2/src/backend/core/providers/parsers/base.py L34.
   - Registered via the eager side-effect import in v2/src/functions/core/parsers/registry.py (L20 `from . import document_intelligence_parser`).
4. Handler orchestration.
   - v2/src/functions/batch_push/handler.py — `batch_push_handler(...)` (approx L72–L112): `content = await download_blob(...)` → `chunks = await parser.parse(content, source=message.filename)`.
5. Parse.
   - v2/src/functions/core/parsers/document_intelligence_parser.py — `DocumentIntelligenceParser.parse` (approx L110–L173):
     - `poller = await client.begin_analyze_document(self._settings.document_intelligence.model_id, AnalyzeDocumentRequest(bytes_source=content))` (approx L112–L116).
     - `except AzureError: logger.exception(...); raise` (approx L117–L128).
     - Page pass: one chunk per page from `page.lines[*].content` (approx L130–L146). **Pageless DOCX → 0 page chunks.**
     - Paragraph fallback: `if not chunks:` → one chunk per `result.paragraphs` (approx L148–L172).
6. Embed + push (only reached if parse returns).
   - v2/src/functions/batch_push/handler.py — `embedder.embed(chunks)` (approx L102), vector-count check (approx L104–L108), `_build_document` (L58–L70), `search_provider.merge_or_upload_documents(...)` (approx L110).

Filename in the poison envelope matches: `{"container_name":"documents","filename":"MSFT_FY23Q4_10K.docx","force_reindex":false}` → extension `docx` → `DocumentIntelligenceParser`.

## Q2 — Paragraph fallback present in current source? YES

v2/src/functions/core/parsers/document_intelligence_parser.py, `parse` (approx L148–L172):

```python
# Office and HTML formats (DOCX, PPTX, XLSX, HTML) are "pageless":
# Document Intelligence returns their text in ``result.paragraphs``
# and leaves ``page.lines`` empty, so the page pass above yields no
# chunks. Fall back to one ``Chunk`` per paragraph ...
if not chunks:
    for paragraph in result.paragraphs or []:
        paragraph_text = (paragraph.content or "").strip()
        if not paragraph_text:
            continue
        chunks.append(
            Chunk(
                id=self.make_chunk_id(source, index),
                content=paragraph_text,
                source=source,
                index=index,
            )
        )
        index += 1

return chunks
```

Class docstring also documents the pageless/paragraph strategy (v2/src/functions/core/parsers/document_intelligence_parser.py, approx L28–L38).

Unit-tested in current source:
- v2/tests/functions/core/parsers/test_document_intelligence_parser.py — `test_parse_falls_back_to_paragraphs_when_pages_have_no_lines` (L297), `test_parse_prefers_pages_and_skips_paragraph_fallback_when_pages_have_text` (L331), `test_parse_paragraph_fallback_skips_empty_and_keeps_indices_dense` (L350), `test_parse_returns_empty_list_when_result_has_no_pages_or_paragraphs` (L284).

**Conclusion:** the BUG-0049 fix is present and covered in current source; the parser code is not the regression.

## Q3 — Root cause: stale-deploy of BUG-0049 vs new bug

**Stale-deploy of BUG-0049 is RULED OUT.** Evidence chain:

- BUG-0049 was a *silent* failure. v2/docs/bugs.md BUG-0049 entry: "the handler `batch_push_handler` then logs its zero-chunk info line and returns `[]` without ever embedding or indexing, and the upload boundary already returned `200` — hence the silent false success." → No dead-letter.
- Current handler still treats 0 chunks as a **warning + `return []`**, deliberately **not** a raise:
  - v2/src/functions/batch_push/handler.py (approx L90–L101):
    ```python
    if not chunks:
        logger.warning(
            "batch_push produced zero chunks",
            extra={ ... },
        )
        return []
    ```
  - v2/src/functions/batch_push/handler.py docstring (approx L38–L48): "It stays a warning, not a raise: a legitimately text-free input (image-only PDF, blank file) also yields zero chunks, and raising would poison-loop it forever."
- Therefore, whether the deployed image is the current parser (paragraph fallback) or a pre-BUG-0049 parser (no fallback), the **empty-chunk path never dead-letters** — it silently succeeds. A stale image would reproduce BUG-0049's *silent* symptom, not BUG-0088's *dead-letter* symptom.

**Because BUG-0088 dead-letters, it is a NEW/DIFFERENT failure: a raised exception in the download → parse → embed → push path.** This is a different defect class than BUG-0049.

(One nuance to note for the writer: BUG-0088's phrase "produced 0 chunks in the index" has two possible mechanisms — (A) parse returns `[]` → silent, no dead-letter; (B) parse THROWS before returning → nothing indexed AND dead-letter. The dead-letter proves mechanism (B). "0 chunks in the index" in BUG-0088 is the *consequence of the throw*, not the BUG-0049 empty-return path.)

## Q4 — What throws for `.docx` but not PDF? (reconcile "dead-letters ⟹ throw")

Dead-letter after 5 retries ⟹ an exception was raised and re-raised on every attempt:
- v2/src/functions/core/exception_mapping.py — `log_queue_errors` (approx L137–L200): `ValidationError` → `logger.warning` + `raise`; `AzureError` → `logger.exception` + `raise`; final `except Exception` → `logger.exception` + `raise`. Docstring: "All three branches **re-raise**. ... the runtime owns the retry policy." So any exception in `_execute` → re-raised → runtime retry → poison.

Where can a throw diverge by format? `.docx` and PDF are **identical** in the code path except for the bytes handed to Document Intelligence:
- Same class/route: both `docx` and `pdf` register to `DocumentIntelligenceParser` (v2/src/functions/core/parsers/document_intelligence_parser.py L59–L64; test asserts both keys resolve to the class — v2/tests/functions/core/parsers/test_document_intelligence_parser.py L64–L69).
- Same `model_id` + `api_version` (from `AppSettings.document_intelligence`; passed at v2/src/functions/core/parsers/document_intelligence_parser.py L96–L100 and L112–L116).
- Same embedder (`embedders_registry.registry.get("azure_openai")`, blueprint.py approx L127) and same search push (`resolve_search_provider`, handler `merge_or_upload_documents`).

So the throw is **inside `begin_analyze_document` for the DOCX bytes**, surfacing as `AzureError` and re-raised:
- v2/src/functions/core/parsers/document_intelligence_parser.py (approx L112–L128).

**Most probable concrete cause — Document Intelligence does not support the Office format in the deployed runtime's api-version/model:**
- Office-format (DOCX/XLSX/PPTX/HTML) support for `prebuilt-layout` requires api-version `2024-02-29-preview`+ (GA `2024-11-30`). PDF is supported at *every* api-version. So an older api-version (or a model that doesn't accept Office, e.g. `prebuilt-read` on older versions) yields a DOCX-only `InvalidRequest` / unsupported-content error while PDF succeeds.
- **Source default is correct** (`api_version = "2024-11-30"`, `model_id = "prebuilt-layout"`): v2/src/backend/core/settings.py L497–L498.
- But both are **operator-pinnable via env**: `env_prefix="AZURE_DOCUMENT_INTELLIGENCE_"` → `AZURE_DOCUMENT_INTELLIGENCE_API_VERSION`, `AZURE_DOCUMENT_INTELLIGENCE_MODEL_ID` (v2/src/backend/core/settings.py L466–L498; docstring L487–L492: "operator-pinnable because GA cuts ... occasionally change default behavior"). A stale/older pin on the deployed Function App (or a deployed image predating the GA-default bump) would produce a DOCX-only throw with an otherwise-correct source tree.

**Why unit tests miss it:** every DI test constructs a mocked client (`_make_fake_client_with_result`, v2/tests/functions/core/parsers/test_document_intelligence_parser.py L54–L61) and hands it a canned `result`; the real service's Office-format acceptance is never exercised. Hence green unit tests + cloud DOCX failure.

**Lower-probability alternates (kept for completeness):**
- DI Office size/limit rejection on this specific `MSFT_FY23Q4_10K.docx` (~1.1 MB — within typical limits, so unlikely).
- Embedder token-ceiling throw on a very large paragraph chunk — but PDF also embeds, empties are stripped, and a parse throw would precede embed. Low probability.
- Vector-count mismatch `RuntimeError` (handler.py approx L104–L108) — format-agnostic, would also hit PDF. Ruled out as DOCX-specific.

## Q5 — Recommended fix + definitive repro/validate

**Definitive repro (capture the REAL error, bypassing the telemetry gap BUG-0055/0089):**
1. Download `MSFT_FY23Q4_10K.docx` from the `documents` container.
2. Run `DocumentIntelligenceParser.parse` against the **live** endpoint using the **deployed** env values (`AZURE_AI_SERVICES_ENDPOINT`, `AZURE_DOCUMENT_INTELLIGENCE_API_VERSION`, `AZURE_DOCUMENT_INTELLIGENCE_MODEL_ID`) — a small opt-in integration script/test — to capture the raised `AzureError` code/message. A single live call reproduces the throw and tells you whether it is an `InvalidRequest`/unsupported-format (api-version/model) vs some other DI error (size/corrupt/password).
   - Re-triggering via `POST /api/admin/documents/reprocess` reproduces the dead-letter but will NOT surface the exception text (App Insights is blind — BUG-0055/0089). The live parse capture is the reliable path.
3. Inspect the deployed config: `azd env get-values` and/or `az functionapp config appsettings list -n func-<SUFFIX> -g <RESOURCE_GROUP>` for `AZURE_DOCUMENT_INTELLIGENCE_API_VERSION` and `AZURE_DOCUMENT_INTELLIGENCE_MODEL_ID`. If pinned to a pre-Office api-version or a non-layout model, that is the root cause.

**Recommended fix (contingent on the captured error):**
- If the deployed api-version/model is stale: ensure the deployed Function App **and** backend use `api_version=2024-11-30` (≥ `2024-02-29-preview`) and `model_id=prebuilt-layout` — i.e., remove the stale env pin so the GA default applies, or wire the GA default in Bicep so it survives re-provision. (Matches the operator-pinnable rationale in settings.py L487–L492.)
- If the captured error is a different DI condition (size/corrupt/password-protected DOCX), handle per that specific error; that would be a genuinely new content-handling case, still distinct from BUG-0049.

**Test file to use/extend:** v2/tests/functions/core/parsers/test_document_intelligence_parser.py.
- `test_parse_wraps_azureerror_with_structured_logger_and_reraises` (L377) already proves the parser re-raises `AzureError` (the mechanism by which DOCX dead-letters).
- Add a **live/integration** repro (opt-in, real endpoint, real sample `.docx`) asserting `parse(...)` returns > 0 chunks — the only test shape that can reproduce a *service* content-support gap, since unit mocks cannot. This is the validation that would definitively confirm/deny the api-version hypothesis and guard the fix.

---

## Evidence index (file:line)

- Queue trigger + `_execute` + parser routing: v2/src/functions/batch_push/blueprint.py (approx L74–L82 filename→key, L93–L145 `_execute`, L155–L169 trigger).
- Extension→key: v2/src/backend/core/paths.py L11–L19.
- `ParserKey.DOCX = "docx"`: v2/src/backend/core/providers/parsers/base.py L34.
- DOCX→`DocumentIntelligenceParser` registration: v2/src/functions/core/parsers/document_intelligence_parser.py L59–L64; eager import v2/src/functions/core/parsers/registry.py L20.
- Handler orchestration + zero-chunk warning-not-raise: v2/src/functions/batch_push/handler.py L38–L48 (docstring), L72–L112 (handler), L90–L101 (zero-chunk `return []`).
- Parse + AzureError re-raise + page pass + paragraph fallback: v2/src/functions/core/parsers/document_intelligence_parser.py L110–L173.
- Queue error re-raise ladder: v2/src/functions/core/exception_mapping.py L137–L200.
- DI settings (defaults + operator-pinnable env): v2/src/backend/core/settings.py L466–L498 (defaults L497–L498).
- DI tests (mocked client, fallback + reraise coverage): v2/tests/functions/core/parsers/test_document_intelligence_parser.py L54–L61 (mock builder), L64–L69 (docx/pdf both route to class), L284/L297/L331/L350 (fallback), L377 (AzureError reraise).
- BUG cross-refs: v2/docs/bugs.md BUG-0088 (L147), BUG-0049 (L898+), BUG-0058 (stale-deploy theme, L117), BUG-0055 (telemetry gap), BUG-0089 (env mis-read → telemetry off).

## Clarifying questions (need runtime/operator input — cannot be answered by source read)

1. What are the *deployed* Function App values of `AZURE_DOCUMENT_INTELLIGENCE_API_VERSION` and `AZURE_DOCUMENT_INTELLIGENCE_MODEL_ID`? (Confirms/denies the top hypothesis outright.)
2. What is the exact `AzureError` code/message returned by `begin_analyze_document` for this DOCX against the live endpoint? (Distinguishes api-version/model vs a content-specific DI error.)

## Recommended next research (not done this session)

- [ ] Run the live-endpoint parse capture on `MSFT_FY23Q4_10K.docx` and record the `AzureError` code/message.
- [ ] Read the deployed Function App app-settings for the two DI env vars; compare to source defaults.
- [ ] Check whether v2/infra/main.bicep wires `AZURE_DOCUMENT_INTELLIGENCE_*` on the Function App / backend (would show if an older value can be pinned at provision).
- [ ] Confirm whether the deployed function image predates the settings.py `2024-11-30` default (relates to BUG-0058 stale-artifact theme).
