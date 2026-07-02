<!-- markdownlint-disable-file -->
# Review Log: BUG-0088 Phase 1 — .docx ingestion root cause + embedder batching fix

## Review Metadata

| Field | Value |
|---|---|
| **Review date** | 2026-07-02 |
| **Reviewer** | Task Reviewer |
| **Plan** | .copilot-tracking/plans/2026-07-02/bug-0088-docx-and-ingestion-validation-plan.instructions.md |
| **Changes log** | .copilot-tracking/changes/2026-07-02/bug-0088-docx-and-ingestion-validation-changes.md |
| **Research** | .copilot-tracking/research/2026-07-02/bug-0088-docx-and-ingestion-validation-research.md |
| **Planning log** | .copilot-tracking/plans/logs/2026-07-02/bug-0088-docx-and-ingestion-validation-log.md |
| **Scope** | Phase 1 (Steps 1.1, 1.2, 1.4 done; 1.3 open pending redeploy). Phases 2-5 not started. |

## Summary of Findings

Phase 1 diagnosis + code fix are **correct and validated at the code level**. The confirm-first probe overturned both research hypotheses with live evidence and landed the right fix (embedder batching) with a test. The fix is **not yet live** — a function redeploy is required (correctly tracked as plan Step 1.3 open). Zero Critical, zero Major; two Minor follow-ups.

| Severity | Count | Notes |
|---|---|---|
| Critical | 0 | — |
| Major | 0 | — |
| Minor | 2 | RV-01 token-limit batching not addressed; RV-02 Step 1.4 details describe a different guard than delivered |
| Info | 3 | not-live-until-redeploy (by design); chunking granularity WI; both plan hypotheses superseded by live evidence |

## Applicable Conventions (applyTo match)

Changed source is `v2/src/backend/core/**` and `v2/tests/backend/**`, so these apply:

* .github/instructions/v2-backend-core.instructions.md
* .github/instructions/v2-tests.instructions.md
* .github/copilot-instructions.md (Hard Rules #1/#2/#11/#14/#16/#17/#18/#19)

## Implementation Review

### Root-cause diagnosis (Steps 1.1-1.2) — VERIFIED

The live probe dead-lettered the `.docx`; a local probe against the same live DI endpoint proved current-source parse = 5047 chunks (no throw) and a local parse+embed reproduced `openai.BadRequestError: array length must be 2048 or less`. This is evidence-based and reproducible. Both research hypotheses (DI api-version throw; stale image) were correctly ruled out (the deployed function uses the `2024-11-30` default with no override; the paragraph fallback is present in source). Recorded as planning-log DD-04.

### The fix (`AzureOpenAIEmbedder.embed`) — VERIFIED

* Batches `chunks` into `_MAX_EMBED_INPUTS` (2048)-sized slices, calls `provider.embed` per batch, and returns one `EmbeddingResult` per batch; the `batch_push` handler already flattens `[v for r in results for v in r.vectors]`, so the multi-result return is consumed correctly (23 batch_push tests pass).
* `2048` is the correct inclusive cap (the API message is "2048 or less").
* Per-batch and total vector-count guards are retained; the `AzureError` handler now carries `batch_start`/`batch_size` structured context (Hard Rule #14).
* `_MAX_EMBED_INPUTS` is a lone module constant (Hard Rule #11 UPPER_SNAKE). Imports at top (#17). Comments describe the technical constraint, not process narrative (#16 gate passes). Test landed same turn (#2).

### Findings

* **RV-01 (Minor) — token-limit batching not addressed.** The fix batches by input **count** (≤2048), which fixes the confirmed array-length 400. Azure OpenAI embeddings also enforce a per-request **token** cap; a batch of 2048 *long* chunks could still 400 on tokens. Not the observed failure (the `.docx` chunks are short paragraphs), so out of scope for the confirmed bug — but recommend a token-aware batch bound as a follow-up for robustness.
* **RV-02 (Minor) — Step 1.4 details/plan text stale vs delivered guard.** The details for Step 1.4 describe an "opt-in live `.docx` parse test (asserts > 0 chunks)"; the delivered guard is instead a unit test of the embedder **batching** (`test_embed_batches_inputs_over_the_array_cap`). The delivered test is the *better* guard (parse was never the failure; the embed batch was), but the plan/details text was not updated to match. Cosmetic doc drift; the changes log + DD-04 explain the deviation.

### Info / notes

* **Not live until redeploy.** The deployed function still has the unbatched embedder; BUG-0088 is code-fixed but not live-verified. Correctly tracked (plan Step 1.3 open; changes log "Pending redeploy").
* **Chunking granularity (WI).** Even with batching, the 10-K yields 5047 one-paragraph chunks (index bloat + retrieval-quality concern). Flagged as a fast-follow decision, not a defect in this fix.
* **Cleanup honored.** Probe blob deleted, stale poison drained, temp scripts removed; all queues at 0.

## Validation Command Outputs

| Command | Result |
|---|---|
| `pytest tests/backend/core/providers/embedders/` + 3 AST gates | **521 passed** |
| `pytest tests/functions/batch_push/` | **23 passed** |
| `pyright --strict` on `azure_openai.py` | **0 errors / 0 warnings** |
| env-ID gate on tracked tree | **pass** (no leaks in this turn's tracking edits) |

## Missing Work and Deviations

* Deviation (intentional, recorded): the fix is embedder batching, not the plan's Step 1.3 DI-api-version change (DD-04). The api-version branch is superseded.
* Remaining Phase 1: Step 1.3 = **function redeploy + live `.docx` re-validation** (operator-confirmed heavier action).
* Phases 2-5 (full file-type matrix, delete contract, pgvector, orchestrators) not started.

## Follow-Up Work

* **Redeploy + live re-validation** (Step 1.3) — required to close BUG-0088 live.
* **RV-01** — token-aware embed batching (robustness).
* **RV-02** — reconcile the Step 1.4 details text with the delivered batching guard (or leave; behavior is correct).
* **WI (granularity)** — group paragraph-fallback chunks into reasonable sizes (index bloat + retrieval quality).
* Phases 2-5 per the plan.

## Overall Status

✅ **Complete (code level) — pending live redeploy.** The BUG-0088 root cause is correctly diagnosed with reproducible live+local evidence, and the embedder batching fix is correct, tested (8 embedder tests incl. the new batching case; 23 batch_push tests; 521 in the reviewed slice), pyright-clean, and Hard-Rule-compliant. Zero Critical / zero Major. Two Minor follow-ups (token batching, Step 1.4 doc drift). The fix is not live until a function redeploy (correctly tracked). Nothing committed (git-ownership).
