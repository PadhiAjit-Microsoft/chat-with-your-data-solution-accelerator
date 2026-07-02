# Research: CWYD v2 Orchestrators × Store Matrix

Status: Complete
Date: 2026-07-02
Scope: READ-ONLY mapping of `agent_framework` + `langgraph` orchestrators across the two
storage modes (cosmosdb + Azure AI Search vs postgresql + pgvector), plus a per-cell
validation plan. No code modified.

## Research questions

1. 2x2 compatibility matrix (agent_framework, langgraph) × (AzureSearch, pgvector): supported vs guarded cells; how retrieval + citations work in each supported cell (file:line).
2. How orchestrator selection happens per deployment (env `CWYD_ORCHESTRATOR_NAME` + db type + any guard), file:line.
3. Per-cell VALIDATION PLAN: exact steps to prove grounded retrieval in each supported cell.
4. Known divergence in citation/reasoning behavior between orchestrators or stores.

---

## 0. Executive summary

- **The store is not independently selectable from the db type.** `DatabaseSettings._enforce_mode_consistency` hard-binds `db_type == cosmosdb ⟺ index_store == AzureSearch` and `db_type == postgresql ⟺ index_store == pgvector` (v2/src/backend/core/settings.py, lines 210-235). So there are exactly **two deployment shapes** — a cosmosdb/AzureSearch deployment and a postgresql/pgvector deployment — and within each, **both** orchestrators are runtime-switchable from the admin UI.
- **All four matrix cells are SUPPORTED. No cell is guarded off today.** The original ADR-0022 guard that rejected `agent_framework` × pgvector with `ConfigResolutionError` → HTTP 409 was **superseded by ADR-0027 / BUG-0066** (2026-06-19). `resolve_effective_config` no longer raises for any orchestrator×store pairing (v2/src/backend/services/admin.py, `resolve_effective_config` ends at `return AdminConfig(**values)` with no raise). The `ConfigResolutionError` primitive + 409 handler still exist as a reusable seam but no rule currently uses them (grep for `raise ConfigResolutionError` / `orchestrator_requires_azure_search` = 0 hits in `v2/src/backend/**`).
- **Retrieval mechanism differs by cell; citation format is shared** (ADR-0026 / Hard Rule #20). Three of four cells run **app-side RAG** (embed query → `BaseSearch.search(vector=…)` → inject `[docN]` block → `filter_to_referenced`). Only `agent_framework` × AzureSearch runs **server-side Foundry IQ Knowledge Base** retrieval (native `【N:M†source】` annotations), which is then normalized to the same `[docN]` + filename shape via `normalize_kb_citations` + `enrich_kb_citations`.

---

## 1. The 2x2 compatibility matrix

Storage is bound to db type (settings coupling above), so the "store" axis is equivalently the "deployment" axis.

| Orchestrator | Store (deployment) | Status | Retrieval path | Citation path |
|---|---|---|---|---|
| `langgraph` | pgvector (postgresql) | **Supported** — postgresql infra default | app-side: embed → `PgVector.search(vector=)` dense cosine | shared `[docN]` block + `filter_to_referenced` |
| `langgraph` | AzureSearch (cosmosdb) | **Supported** — runtime-switchable | app-side: embed → `AzureSearch.search(vector=)` hybrid + semantic re-ranker | shared `[docN]` block + `filter_to_referenced` |
| `agent_framework` | AzureSearch (cosmosdb) | **Supported** — cosmosdb infra default (ADR-0021) | server-side Foundry IQ KB MCP tool | native annotations → `normalize_kb_citations` → `enrich_kb_citations` → `[docN]` + filename |
| `agent_framework` | pgvector (postgresql) | **Supported** — runtime-switchable (ADR-0027 / BUG-0066) | app-side: embed → `PgVector.search(vector=)` dense cosine, injected into **user turn** | shared `[docN]` block + `filter_to_referenced` |

**No guarded-off cell.** ADR-0027 decision #3: "The pgvector + `agent_framework` config rule is removed. `resolve_effective_config` no longer raises `ConfigResolutionError` for that pairing. Every orchestrator / index-store cell is now served."

### Cell A — `langgraph` × pgvector (postgresql default)

- Registration: `@registry.register("langgraph")` at v2/src/backend/core/providers/orchestrators/langgraph.py line 72.
- `run()` embeds the latest user text, then searches with the vector:
  - `embedding = await self.llm.embed([query])` (langgraph.py ~line 183)
  - `query_vector = embedding.vectors[0] if embedding.vectors else None` (~line 185)
  - `sources = await self._search.search(query, top_k=…, use_semantic_search=…, vector=query_vector)` (~line 189)
- `PgVector.search` with a `vector` runs **dense cosine**: `1 - (content_vector <=> $1::vector) AS score … ORDER BY content_vector <=> $1::vector LIMIT $2` (v2/src/backend/core/providers/search/pgvector.py lines ~112-121). `use_semantic_search` is a **no-op** on pgvector (pgvector.py lines 103-107, `_ = use_semantic_search`).
- **BUG-0065 (fixed 2026-06-19)** is exactly this: before the fix `langgraph` called `search()` **without** `vector=`, and pgvector's no-vector fallback is Postgres FTS `plainto_tsquery('english', $1)` which **AND-joins every lexeme** (pgvector.py lines ~127-135) → 0 rows for a natural-language question → out-of-domain fallback. The `vector=` addition made dense cosine the path.
- Citations: `citations = build_citations(sources)`, `block = format_sources_block(sources)` injected as a **system** message `Sources:\n{block}` (langgraph.py ~lines 195-199); after the answer, `filter_to_referenced(answer, citations)` emits one `citation` event per marker the model actually used (~lines 240-245), then a single `answer` event (~line 246).

### Cell B — `langgraph` × AzureSearch (cosmosdb)

- Same `langgraph.run()` code path as Cell A; only the injected `BaseSearch` differs (registry-dispatched by `index_store`).
- `AzureSearch.search` with a `vector` runs **hybrid** (text + vector): `vector_queries=[VectorizedQuery(vector=…, k_nearest_neighbors=effective_top_k, fields="content_vector")]` (v2/src/backend/core/providers/search/azure_search.py lines 162-171), and when `use_semantic_search` is on it adds `query_type=QueryType.SEMANTIC` + `semantic_configuration_name="default"` (azure_search.py lines ~175-177).
- BUG-0065 note: AzureSearch **masked** the missing-`vector` caller bug because its no-vector path is full-text **plus the Azure semantic re-ranker**, which still surfaced relevant chunks; pgvector had no reranker so it broke. Post-fix, both stores get `vector=`, so AzureSearch is now true hybrid.
- Citations: identical shared seam (`build_citations` / `format_sources_block` / `filter_to_referenced`).

### Cell C — `agent_framework` × AzureSearch (cosmosdb default, ADR-0021)

- Registration: `@registry.register("agent_framework")` at v2/src/backend/core/providers/orchestrators/agent_framework.py line 101.
- `run()` builds the KB tool: `kb_tool = self._build_kb_tool()` (agent_framework.py ~line 178). `_build_kb_tool()` returns an `MCPTool` bound to `{endpoint}/knowledgebases/{kb_name}/mcp?api-version=…` with `require_approval="never"`, `allowed_tools=[KB_RETRIEVE_TOOL_NAME]` (`"knowledge_base_retrieve"`, agent_framework.py line ~96), and `project_connection_id=self._connection_name` (server-side auth via the project search connection). Returns `None` when endpoint/kb_name/connection_name are empty (agent_framework.py `_build_kb_tool`, endpoint guard).
- The tool is passed as `extra_tools=[kb_tool.as_dict()]` to `agents.build_agent(CWYD_AGENT, db, extra_tools=…)`; the Foundry Responses model performs retrieval **server-side** and emits **native** citation annotations.
- Citation normalization (agent_framework.py, post-answer block): `citations = citations_from_annotations(citation_annotations)` → `answer, citations = normalize_kb_citations(answer, citations)` (rewrites native `【N:M†source】` markers to grouping-ordered `[docN]`, offset-anchored via `metadata["annotated_regions"]`) → `enrich_kb_citations(citations, search.get_document_by_key)` backfills friendly `title` / `snippet` / `url` because a KB annotation carries only the raw `mcp://searchindex/<key>` id.
- The injected `search` provider here is the **AzureSearch** handler, used only for `get_document_by_key` enrichment (agent_framework.py constructor comment: "In practice this is the AzureSearch handler").

### Cell D — `agent_framework` × pgvector (postgresql, ADR-0027 / BUG-0066)

- Same `run()`; the branch is gated on `_build_kb_tool()` returning `None` (no KB in pgvector mode) **and** a wired `BaseSearch`:
  - `if kb_tool is None and self._search is not None:` (agent_framework.py ~line 197)
  - embed → `query_vector` → `sources = await self._search.search(query, top_k=…, use_semantic_search=…, vector=query_vector)` (agent_framework.py ~lines 210-218) — the **PgVector** dense-cosine path, identical to Cell A.
  - `retrieved_citations = build_citations(sources)`; `block = format_sources_block(sources)` **prepended to the latest USER turn** (not a system message), because the Agents Responses thread drops system messages (`_to_oss_messages` only forwards `user`/`assistant`) (agent_framework.py ~lines 219-233).
- Citation branch after the answer: `if retrieved_citations:` → `citations = filter_to_referenced(answer, retrieved_citations)` (agent_framework.py ~post-answer) — the **same shared seam** as `langgraph`; no native-annotation normalization or enrichment on this path.
- ADR-0027 consequence: on pgvector, `agent_framework` does **not** use Foundry IQ (impossible — Foundry IQ has no pgvector knowledge-source type); its value-add there is the Agent runtime (tool-calling, agent thread, reasoning summary), not KB retrieval.

---

## 2. Orchestrator selection per deployment (env + db type + guard)

### 2.1 Registry dispatch (Hard Rule #4, no if/elif)

- The registry instance is a `Registry[Callable[..., OrchestratorBase]]` in v2/src/backend/core/providers/orchestrators/_instance.py.
- Eager side-effect imports populate it: `from . import agent_framework` + `from . import langgraph` at v2/src/backend/core/providers/orchestrators/registry.py lines 30-31 (fires each `@registry.register(...)`). Third-party plugins load via `load_entry_points("cwyd.providers.orchestrators")` (registry.py line ~37).
- Registered keys MUST equal `OrchestratorName` values; enforced by `test_router_uses_registry_dispatch_no_hardcoded_provider_names` (registry.py docstring lines ~13-19).

### 2.2 The registry keys (StrEnum)

- `OrchestratorName(StrEnum)` at v2/src/backend/core/settings.py line 90: `LANGGRAPH = "langgraph"`, `AGENT_FRAMEWORK = "agent_framework"`.

### 2.3 The env default (`CWYD_ORCHESTRATOR_NAME`)

- `OrchestratorSettings` at settings.py line 353; `model_config = SettingsConfigDict(env_prefix="CWYD_ORCHESTRATOR_", extra="ignore")` line 372; field `name: OrchestratorName | str = OrchestratorName.AGENT_FRAMEWORK` line 388. So the **code default** is `agent_framework` (ADR-0021).
- The `OrchestratorName | str` widening is the Hard Rule #11 registry-driven carve-out (settings.py lines 374-386): the `str` arm admits third-party keys registered against `cwyd.providers.orchestrators`; validation moves to the registry `.get(...)` boundary.

### 2.4 Per-db-type wiring in infra (BUG-0064)

- **Bicep sets the env var per db type**: v2/infra/main.bicep line 1879:
  `{ name: 'CWYD_ORCHESTRATOR_NAME', value: databaseType == 'postgresql' ? 'langgraph' : 'agent_framework' }`
  (compiled into v2/infra/main.json line ~48372).
- **BUG-0064 (fixed 2026-06-17)** was two faults: (a) bicep previously emitted `ORCHESTRATOR` (not `CWYD_ORCHESTRATOR_NAME`), so the value was never read and fell to the `agent_framework` code default; (b) the value was hardcoded `agent_framework` for every db type, but pgvector's only coherent default at that time (pre-ADR-0027) was `langgraph`. Fix = the per-db-type ternary above.
- Net effect: a **postgresql** deployment boots defaulting to `langgraph`; a **cosmosdb** deployment boots defaulting to `agent_framework`. Both are then switchable at runtime.

### 2.5 Runtime resolution + admin override (per request)

- v2/src/backend/routers/conversation.py, `conversation(...)`:
  - `effective = resolve_effective_config(settings, overrides)` (~line 83), `orchestrator_name = effective.orchestrator_name` (~line 86). This overlays a persisted `RuntimeConfig.orchestrator_name` (admin-saved, live-reloaded onto `app.state.runtime_overrides`) on the `CWYD_ORCHESTRATOR_NAME` env default.
  - Single registry dispatch: `orchestrator = orchestrators_registry.registry.get(orchestrator_name)(settings=…, llm=…, search=…, agents=…, db=…, credential=…, agent_name=CWYD_AGENT.name, system_prompt=effective.cwyd_agent_instructions, search_top_k=…, search_use_semantic_search=…, openai_temperature=…, openai_max_tokens=…)` (conversation.py ~lines 113-129). Every orchestrator gets the **same uniform kwargs** and swallows the rest via `**_extras` (Hard Rule #4).
- `resolve_effective_config` is at v2/src/backend/services/admin.py; it builds a `values` dict (orchestrator_name = `settings.orchestrator.name`, etc.), overlays non-None overrides, re-wraps `cwyd_agent_instructions` through `resolve_cwyd_instructions`, and `return AdminConfig(**values)` — **no orchestrator×store guard raise**.

### 2.6 The (now-dormant) guard

- ADR-0022 originally raised `ConfigResolutionError(reason="orchestrator_requires_azure_search")` at the resolver when `index_store == pgvector` and `orchestrator == agent_framework`, mapped to HTTP 409 by an app-level handler (v2/src/backend/exception_handlers.py line ~40: `ConfigResolutionError → 409`).
- ADR-0027 (2026-06-19) **superseded decisions #2 and #5** of ADR-0022. The pgvector guard is removed; the `ConfigResolutionError` class (v2/src/backend/services/admin.py line ~79) + the 409 handler survive as a **reusable seam for any FUTURE incompatibility** but are currently unused (0 `raise ConfigResolutionError` sites in `v2/src/backend/**`).
- `OrchestratorSettings.name` docstring at settings.py lines ~384-388 still says pgvector + `agent_framework` "is rejected at request time with a `ConfigResolutionError` (HTTP 409) per ADR 0022" — this is **stale text** (ADR-0027 Follow-ups explicitly flags this and Hard Rule #20 R3 text as needing an update). Treat the code (ADR-0027 behavior) as authoritative, not the docstring.

---

## 3. Per-cell validation plan

**Preconditions (all cells).** A grounded corpus must be indexed. The canonical live-tested doc set in `bugs.md` is `Benefit_Options.pdf`, `PerksPlus.pdf` (cosmosdb/AzureSearch — BUG-0059) and a small pgvector chunk set (BUG-0065: 6 chunks, all `vector_dims=1536`). Admin orchestrator switch is `PATCH /api/admin/config` `{ "orchestrator_name": "<key>" }`; the effective value is confirmed via `GET /api/admin/config/effective` and `GET /api/admin/status` (BUG-0068 made `/status` overlay the override).

**Shared assertions (every supported cell).** For a grounded question, `POST /api/conversation` (SSE `Accept: text/event-stream`) must yield, in order:
1. one leading `reasoning` frame with `metadata.placeholder=true` carrying `KB_SEARCH_NARRATION` ("Searching the knowledge base for relevant sources…") — emitted by `run_chat` only when `search is not None` (conversation.py `retrieval_hint` gate; pipelines/chat.py `KB_SEARCH_NARRATION`);
2. ≥1 `citation` frame whose `metadata.id` matches `^\[doc\d+\]$` and whose `title` is the source **filename** (not a raw `mcp://…` id);
3. exactly one buffered `answer` frame containing the matching inline `[docN]` marker(s);
4. a terminal `conversation` control frame carrying the resolved `conversation_id`.
Buffered mode (default `Accept`) must return `ConversationResponse{ content, citations[], conversation_id }` with the same `[docN]` markers + filename citations.

**Negative assertion (every cell).** An out-of-domain question (e.g. "write a poem about a dragon in France") must return the fixed fallback string "The requested information is not available in the retrieved data. Please try another query or topic." (from `CWYD_GUARDRAIL`, definitions.py) and **no** `citation` frame.

### Cell A — `langgraph` × pgvector (postgresql deployment default)

1. Deploy/point at a postgresql env; confirm `GET /api/admin/status` → `orchestrator_name: "langgraph"` (bicep default) and the health probe is `pass`.
2. Ground question (BUG-0065 canonical): "What is the Contoso remote work policy and how many days per week can I work remotely?"
3. Assert the shared assertions above. Specifically prove the **vector** path: a natural-language multi-term question must ground (pre-BUG-0065 it returned 0 rows via `plainto_tsquery` AND-join → fallback). Expect `[doc1]` + the source filename, dense-cosine retrieval.
4. Direct DB sanity (optional): the store has `content_vector vector(1536)` rows; `plainto_tsquery` on the full question → 0, but the embedded-vector cosine query → ≥1 (BUG-0065 evidence).

### Cell B — `langgraph` × AzureSearch (cosmosdb deployment, switched)

1. On a cosmosdb env, `PATCH /api/admin/config {"orchestrator_name":"langgraph"}`; confirm `/config/effective` + `/status` show `langgraph`.
2. Ground question: "tell me about employee benefits" (BUG-0028 canonical in-domain probe).
3. Assert shared assertions; expect `[doc1]` + `Benefit_Options.pdf`. This exercises hybrid (text+vector) + semantic re-ranker (`use_semantic_search=true` default).
4. Regression guard (BUG-0028): confirm the softened guardrail does NOT over-refuse — "employee benefits" and "health plans" must ground, while France + dragon-poem must refuse.

### Cell C — `agent_framework` × AzureSearch (cosmosdb deployment default, ADR-0021)

1. On a cosmosdb env, confirm `/status` → `orchestrator_name: "agent_framework"` (bicep default).
2. Precondition (BUG-0059): the backend `AZURE_AI_SEARCH_CONNECTION_NAME` must point at the RemoteTool connection `cwyd-kb-mcp` (audience `https://search.azure.com`), not the bare `CognitiveSearch` connection, or KB grounding 401s.
3. Ground question: "tell me about the perks plus program" (or "employee benefits").
4. Assert shared assertions **plus** the server-side KB signals: a `tool` frame `knowledge_base_retrieve` (`mcp_server_tool_call`) precedes the citations; the answer's native `【N:M†source】` markers are rewritten to `[docN]` (`normalize_kb_citations`) and the citation `title` is the enriched filename (`enrich_kb_citations`, BUG-0030). Confirm no `【…†source】` marker leaks into the `answer` or into any `reasoning` frame (BUG-0043: `strip_kb_markers` on the reasoning channel). Confirm `Citation.url` is `""`, never the literal string `"None"` (BUG-0033).
5. Reasoning path (BUG-0035/BUG-0013): if the answer deployment is a gpt-5/o-series reasoning model, confirm the run streams a reasoning summary and does NOT 400 on `temperature` (reasoning branch omits sampling knobs).

### Cell D — `agent_framework` × pgvector (postgresql deployment, switched — ADR-0027 / BUG-0066)

1. On a postgresql env, `PATCH /api/admin/config {"orchestrator_name":"agent_framework"}`; confirm `/config/effective` + `/status` show `agent_framework` and the request is **not** rejected with 409 (proves the ADR-0022 guard is gone).
2. Ground question (BUG-0066 canonical, same as Cell A).
3. Assert shared assertions; expect a reasoning summary + **pgvector dense-retrieval** citations + a grounded `[docN]` answer, with the citation grounded via the app-side `build_citations` / `filter_to_referenced` seam (NOT native KB annotations — there is no KB in pgvector mode). Verify the `[docN]` sources block was injected into the **user turn** (grounding still works despite system messages being dropped).
4. Frontend regression (BUG-0067): confirm a non-2xx `/api/conversation` (if ever triggered) renders inline, does not blank the page (error boundary + `streamChat` body parsing).

---

## 4. Known divergence between orchestrators / stores

### 4.1 Retrieval mechanism divergence (by cell — intentional)

- **Server-side (Cell C only):** `agent_framework` × AzureSearch grounds through the Foundry IQ Knowledge Base MCP tool (retrieval runs in the Responses API under the project connection identity). All other cells run **app-side** RAG (`BaseSearch.search(vector=…)`).
- **Grounding injection site differs on `agent_framework`:** `langgraph` injects the `[docN]` block as a **system** message (langgraph.py ~line 199); `agent_framework`'s pgvector path prepends it to the **user** turn (agent_framework.py ~lines 224-233) because the Agents Responses thread drops system messages (`_to_oss_messages`).
- **pgvector vs AzureSearch retrieval quality:** AzureSearch supports hybrid + a semantic re-ranker (`use_semantic_search`); pgvector supports only dense cosine or FTS and treats `use_semantic_search` as a no-op (pgvector.py lines 103-107). This is why the missing-`vector` bug (BUG-0065) broke pgvector but not AzureSearch.

### 4.2 Citation shape divergence — CONVERGED (was BUG-0030, fixed)

- Historically the two paths emitted structurally different citations: `langgraph` → `[docN]` + filename; `agent_framework` (KB) → native `【N:M†source】` + raw `mcp://searchindex/<key>`. **BUG-0030 (fixed 2026-06-15)** converged them: `agent_framework` post-processes native annotations through `normalize_kb_citations` (marker → `[docN]`, offset-anchored) + `enrich_kb_citations` (raw id → filename). Net: **all four cells now emit one `[docN]` + filename `Citation` shape** (ADR-0026 / Hard Rule #20 R2, single formatter in `tools/citations.py`).
- Residual `agent_framework`-only cleanup: `_clean_annotation_field` coerces a `None` / `"None"` annotation `url` to `""` (BUG-0033); `langgraph` never has this problem (it builds citations from `SearchResult`, not SDK annotations).

### 4.3 Reasoning-channel divergence — CONVERGED (was BUG-0043, fixed)

- `agent_framework` KB answers can carry native `【N:M†source】` markers inside `text_reasoning` blocks; the reasoning panel has no `[docN]` rendering. **BUG-0043 (fixed 2026-06-19)**: `strip_kb_markers(text)` is applied to `text_reasoning` in `_update_to_events` so the reasoning channel drops markers while the answer-side `normalize_kb_citations` owns rewriting them. `langgraph` emits `[docN]` (never native markers), so it is unaffected.
- Both paths surface reasoning-model summaries on the same `reasoning` channel; both gate reasoning knobs on `llm.supports_reasoning()` and both omit `temperature`/`max_tokens` on the reasoning branch (BUG-0035 for `agent_framework`; `langgraph` `reason()` sends neither).

### 4.4 Prompt / guardrail divergence — CONVERGED (was BUG-0031/0032, fixed)

- Both orchestrators resolve grounding instructions through the single seam `resolve_cwyd_instructions(override_text)` → `compose_cwyd_instructions` → fixed `CWYD_GUARDRAIL` appended once (v2/src/backend/core/agents/definitions.py). `langgraph` injects `effective.cwyd_agent_instructions` (already guardrail-wrapped by the resolver) as its system prompt; `agent_framework` wraps independently in `_resolve_definition` via `build_agent`. **BUG-0031** fixed the earlier gap where `langgraph` dropped the guardrail on operator overrides; **BUG-0032** fixed a double-wrap. The `[doc+index]` citation-format directive lives once in `CWYD_GUARDRAIL` (definitions.py, "You **must cite** every claim using the citation format [doc+index]…").

### 4.5 Narration / placeholder behavior — shared

- The during-the-wait `KB_SEARCH_NARRATION` reasoning frame (marked `metadata.placeholder=true`) is emitted **orchestrator-agnostically** by `run_chat` (pipelines/chat.py), gated in the router on `search is not None` (conversation.py `retrieval_hint`). BUG-0036 routes the placeholder into a separate FE slot so a reasoning-capable model replaces it the moment its first native reasoning frame arrives; a non-reasoning model keeps it as the sole panel content.

---

## 5. File:line evidence index

- Orchestrator ABC + channel set: v2/src/backend/core/providers/orchestrators/base.py (class `OrchestratorBase`, abstract `run() -> AsyncIterator[OrchestratorEvent]`).
- `langgraph` registration + retrieval + citations: v2/src/backend/core/providers/orchestrators/langgraph.py (register line 72; embed ~183; `search(vector=)` ~189; `build_citations`/`format_sources_block` ~195-199; `filter_to_referenced` ~240-245).
- `agent_framework` registration + dual path: v2/src/backend/core/providers/orchestrators/agent_framework.py (register line 101; `_build_kb_tool()` KB-vs-None; run() `kb_tool` ~178; app-side pgvector branch `if kb_tool is None and self._search is not None` ~197-233; KB citation normalize/enrich post-answer; `KB_RETRIEVE_TOOL_NAME` ~96).
- Registry instance + eager imports: v2/src/backend/core/providers/orchestrators/_instance.py; v2/src/backend/core/providers/orchestrators/registry.py (imports lines 30-31; entry-points ~37).
- `OrchestratorName` StrEnum: v2/src/backend/core/settings.py line 90.
- `OrchestratorSettings` (`CWYD_ORCHESTRATOR_` env, default `AGENT_FRAMEWORK`): settings.py lines 353-388.
- Store coupling (`_enforce_mode_consistency`): settings.py lines 210-235; `db_type`/`index_store` defaults lines 196-197.
- Router selection + registry dispatch: v2/src/backend/routers/conversation.py (`resolve_effective_config` ~83-86; `orchestrators_registry.registry.get(...)` ~113-129; `retrieval_hint` gate; `run_chat`).
- `resolve_effective_config` (no guard raise): v2/src/backend/services/admin.py (`ConfigResolutionError` class ~79; resolver ends `return AdminConfig(**values)`).
- Bicep per-db-type env: v2/infra/main.bicep line 1879 (`CWYD_ORCHESTRATOR_NAME` ternary); compiled v2/infra/main.json ~48372.
- Shared citation seam: v2/src/backend/core/tools/citations.py (`build_citations`, `format_sources_block`, `filter_to_referenced`, `citations_from_annotations`, `normalize_kb_citations`, `enrich_kb_citations`, `strip_kb_markers`).
- Prompt seam: v2/src/backend/core/agents/definitions.py (`CWYD_GUARDRAIL`, `compose_cwyd_instructions`, `resolve_cwyd_instructions`, `CWYD_AGENT`).
- Pipeline narration + post-prompt: v2/src/backend/core/pipelines/chat.py (`KB_SEARCH_NARRATION`, `run_chat`).
- Search providers: v2/src/backend/core/providers/search/pgvector.py (`search` lines 94-167; dense-cosine vs FTS branch; `use_semantic_search` no-op 103-107); v2/src/backend/core/providers/search/azure_search.py (`search` lines 133-…; `VectorizedQuery` 162-171; semantic re-rank; `get_document_by_key` line 296).
- Exception handler (dormant 409): v2/src/backend/exception_handlers.py line ~40.
- Bugs/ADRs: v2/docs/bugs.md (BUG-0028/0030/0031/0033/0035/0036/0043/0059/0064/0065/0066/0067/0068); v2/docs/adr/0021, 0022, 0026, 0027.

---

## 6. Clarifying questions / caveats

- **Stale docstring caveat (not a defect to fix in this task):** `OrchestratorSettings.name` docstring (settings.py ~384-388) and Hard Rule #20 R3 text still describe the pgvector + `agent_framework` cell as "rejected at config (409, ADR 0022)". ADR-0027 Follow-ups already flags these as pending governing-instruction updates. Validation must assert the cell is **served**, not rejected.
- **Two deployments, not four independent stores:** because of the settings coupling, you cannot validate `langgraph` × AzureSearch on a postgresql deployment (or `agent_framework` × pgvector on a cosmosdb deployment). Each deployment exercises its own store; the orchestrator is the only runtime-switchable axis. Plan for two deployment targets (one cosmosdb, one postgresql), switching orchestrators within each.

## 7. Recommended next research (not done here)

- [ ] Confirm the live cloud `AZURE_AI_SEARCH_CONNECTION_NAME` value on the current cosmosdb deployment (BUG-0059 durable bicep back-port was "pending"; a wrong value 401s Cell C at KB grounding).
- [ ] Verify whether the stale `OrchestratorSettings.name` docstring + Hard Rule #20 R3 have since been updated (out of scope; would need a separate governing-instruction diff).
- [ ] Inspect the frontend SSE consumer to confirm it renders `[docN]` superscripts + the citation detail panel identically for both orchestrators (BUG-0016/0039 territory) — not required for backend grounding proof but relevant for end-to-end UI validation.
