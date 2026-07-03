<!-- markdownlint-disable-file -->
# Implementation Details: CWYD v2 — First `azd up` Deploy Path

## Context Reference

Sources:
* .copilot-tracking/research/2026-07-01/v2-first-azd-up-deploy-research.md — primary synthesis (deploy path, grounding, validation).
* .copilot-tracking/research/subagents/2026-07-01/v2-wi07-ts6133-fix-research.md — WI-07 exact fix.
* .copilot-tracking/research/subagents/2026-07-01/v2-deploy-path-research.md — azd flow, quota, health, gates.
* .copilot-tracking/plans/logs/2026-07-01/v2-containerize-services-and-model-cleanup-log.md — WI-01…WI-09 deferred items.

Starting state: containerize + model-cleanup implementation complete + green (bicep 0 errors, pyright 0/0/0, pytest 2686, vitest 594). No `azd up` yet. Default DB mode `cosmosdb`; region default `eastus2`. azd `1.27.0`.

Placeholder convention (Hard Rule #18 — never real IDs): `<AZURE_SUBSCRIPTION_ID>`, `<AZURE_TENANT_ID>`, `<RESOURCE_GROUP>`, `<AZD_ENV_NAME>`, `<SUFFIX>`, `<REGION>`. Real values come from `azd env get-values`.

---

## Implementation Phase 1: Unblock the frontend production build (WI-07)

<!-- parallelizable: false -->

Single code unit (uncomment one JSX render block) with an existing failing test that turns green — satisfies the CWYD test-first contract without authoring a new test. This must land before any docker/deploy step because the frontend prod image build runs `npm run build`.

### Step 1.1: Uncomment the "Updated by" audit render in Configuration.tsx

Delete the JSX comment wrapper around the 6-line "Updated by" audit block so `formatActor` (declared at line 628) becomes used and `TS6133` clears.

Files:
* v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx - remove the `{/* ` at the head of line 1040 and the ` */}` at the tail of line 1045 (exact lines verified at implementation time — re-read 1030-1055 first, as line numbers drift). The block renders `formatActor(state.lastRuntime.updated_by)` on the audit footer. No other edit: `formatActor` stays; `RuntimeConfig.updated_by` already exists at v2/src/frontend/src/models/admin.tsx:278 (non-null string).

Discrepancy references:
* Resolves WI-07 (planning log v2-containerize…-log.md). Confirmed self-contained — #35d is cleared (DR-DEPLOY-02).

Success criteria:
* From v2/src/frontend/: `npm run build` (or `npx tsc -b`) completes with no `TS6133` (no `'formatActor' is declared but never read`).
* No new TypeScript/lint errors introduced elsewhere.

Context references:
* .copilot-tracking/research/subagents/2026-07-01/v2-wi07-ts6133-fix-research.md (Recommended fix — UNCOMMENT) - exact edit site + data-safety.

Dependencies:
* None (isolated frontend edit).

### Step 1.2: Verify the coupled vitest audit-footer test passes

The pre-existing failing test asserts the restored render; confirm it now passes (test-first closure).

Files:
* v2/tests/frontend/pages/admin/Configuration/Configuration.test.tsx - test at line 505 `"surfaces the audit footer with the runtime updated_at / updated_by metadata after save"`; coupling assertion line 525 `expect(footer).toHaveTextContent("admin-user-id")`. No edit — read-only verification the render satisfies it.

Success criteria:
* From v2/tests/frontend/: `npx vitest run pages/admin/Configuration/Configuration.test.tsx -t "surfaces the audit footer"` passes.
* Full frontend suite from v2/: `npm test` → prior 594 passed becomes 595 passed (the one pre-existing WI-07 fail flips green), 0 failed.

Context references:
* .copilot-tracking/research/subagents/2026-07-01/v2-wi07-ts6133-fix-research.md (Exact failing vitest test).

Dependencies:
* Step 1.1 completion.

### Step 1.3: Validate frontend build + tests (phase gate)

Run the frontend build + full vitest once to confirm the image-build prerequisite is satisfied and nothing else regressed.

Validation commands:
* `npm run build` (from v2/src/frontend/) — frontend production bundle builds clean.
* `npm test` (from v2/) — full vitest suite green (595 passed / 0 failed expected).

---

## Implementation Phase 2: Pre-deploy verification (read-only gates)

<!-- parallelizable: false -->

Read-only checks that must pass before spending a provision. Sequenced after Phase 1 because the frontend docker build (Step 2.3) depends on the WI-07 fix. Steps 2.1/2.2 are independent and may run in any order; Step 2.3 depends on Phase 1; Step 2.4 needs an azd env.

### Step 2.1: Verify model quota in the target region (WI-01)

Confirm GlobalStandard `gpt-5.1` and Standard `text-embedding-3-large` capacity exist in `<REGION>` (default `eastus2`) on the new tenant before provision.

Files:
* v2/infra/main.parameters.json - confirm the model names/versions/skus/capacities (chat `gpt-5.1` v`2025-11-13` GlobalStandard cap 150; embedding `text-embedding-3-large` v`1` Standard cap 100).

Commands (read-only; placeholders only):
* `az cognitiveservices usage list --location <REGION> --query "[?contains(name.value, 'OpenAI.GlobalStandard.gpt-5.1')].{name:name.value,current:currentValue,limit:limit}" -o table`
* `az cognitiveservices usage list --location <REGION> --query "[?contains(name.value, 'OpenAI.Standard.text-embedding-3-large')].{name:name.value,current:currentValue,limit:limit}" -o table`

Success criteria:
* Both models show `limit - current >= required capacity` (150 chat / 100 embedding) in `<REGION>`. If not, request a quota increase or pick another allowed region (`australiaeast`/`eastus2`/`japaneast`/`uksouth`) before proceeding.

Discrepancy references:
* Resolves WI-01 / DR-08.

Dependencies:
* Live `az login` on the target subscription/tenant.

### Step 2.2: Confirm azd version + login (WI-02)

Files:
* (none)

Commands (read-only):
* `azd version` — confirm `>= 1.18.0` and `!= 1.23.9` (observed `1.27.0` ✓).
* `azd auth login --check-status` (or `azd auth login` if not authenticated).

Success criteria:
* azd version satisfies the pin; azd is authenticated against the target tenant. Unique per-build container tags are the default for `host: containerapp` (fresh code every deploy).

Discrepancy references:
* Resolves WI-02.

Dependencies:
* None.

### Step 2.3: Local docker build of all three images

Prove the three Dockerfiles build before handing them to azd/ACR.

Files:
* v2/docker/Dockerfile.backend, v2/docker/Dockerfile.frontend, v2/docker/Dockerfile.functions - build targets confirmed in v2/azure.yaml docker blocks.

Commands (from v2/, context `.`):
* `docker build -f docker/Dockerfile.backend  -t cwyd-backend:local .`
* `docker build -f docker/Dockerfile.frontend --target prod -t cwyd-frontend:local .`
* `docker build -f docker/Dockerfile.functions -t cwyd-function:local .`

Success criteria:
* All three images build with exit 0. The frontend build (which runs `npm run build`) succeeds — proving the Phase 1 fix cleared the ACR-side `tsc` step too.

Dependencies:
* Phase 1 (frontend build fix). Docker daemon available.

### Step 2.4: what-if / preview against the target resource group

Preview the infra delta before provisioning. `main.bicep` is `resourceGroup`-scoped.

Files:
* v2/infra/main.bicep (targetScope resourceGroup, line 25), v2/infra/main.parameters.json.

Commands:
* Preferred: `azd provision --preview` (after Step 3.1 env setup — uses azd parameter binding).
* Or: `az deployment group what-if -g <RESOURCE_GROUP> -f v2/infra/main.bicep -p v2/infra/main.parameters.json` (do NOT use `az deployment sub what-if`).

Success criteria:
* what-if returns a clean create/modify plan with no errors; the resource set matches expectation (backend/frontend/function Container Apps, ACR, Foundry, Search+KB in cosmos mode, storage, no App Service Plan / no Flex Function App).

Dependencies:
* An azd env (Step 3.1) for `azd provision --preview`; or a pre-created target RG for `az deployment group what-if`.

---

## Implementation Phase 3: Provision + deploy (`azd up`)

<!-- parallelizable: false -->

Operational execution. Runs `azd up` end-to-end (provision infra + build/push/deploy all three container images + postprovision KB seed + postdeploy sample-data upload). This is the plan's execution phase; the agent runs it interactively with the operator (login + subscription selection are human-gated).

### Step 3.1: Create/select the azd environment + set parameters

Files:
* v2/azure.yaml (parameters block), v2/infra/main.parameters.json.

Commands (from v2/):
* `azd env new <AZD_ENV_NAME>` (or `azd env select <AZD_ENV_NAME>`).
* `azd env set AZURE_ENV_DATABASE_TYPE cosmosdb` (decision #3 default — cosmosdb also deploys Search + Foundry IQ KB).
* `azd env set AZURE_LOCATION <REGION>` and `azd env set AZURE_ENV_AI_SERVICE_LOCATION <REGION>` (default `eastus2`).
* (optional) leave the 4 WAF booleans at `false` for the default public profile.

Success criteria:
* `azd env get-values` shows `AZURE_ENV_DATABASE_TYPE=cosmosdb`, a valid `AZURE_LOCATION` from the allowed set, and the target subscription bound.

Dependencies:
* Phase 2 gates green.

### Step 3.2: Run `azd up`

Files:
* (none — orchestrated by v2/azure.yaml)

Commands (from v2/):
* `azd up` — provisions infra, builds+pushes the three images to the provisioned ACR, deploys the three Container Apps, runs `postprovision` (index schema + Foundry IQ KB seed) then `postdeploy` (sample-data upload → `documents` container + ingestion enqueue).

Success criteria:
* `azd up` completes with `SUCCESS`. No hook (postprovision/postdeploy) errors. Deployment outputs printed (`AZURE_BACKEND_URL`, `AZURE_FUNCTION_APP_URL`, endpoints).

Dependencies:
* Step 3.1. Deployer holds Owner / User-Access-Administrator (role assignments in main.bicep). Quota confirmed (Step 2.1).

### Step 3.3: Confirm hooks seeded index + sample data

Files:
* v2/scripts/post_provision.py (index + KB seed), v2/scripts/upload_sample_data.py (sample-data upload).

Commands (read-only):
* Inspect `azd up` output for the postprovision KB-seed log lines and the postdeploy upload summary.
* (optional) confirm blobs: `az storage blob list --account-name <SAMPLE_STORAGE_ACCOUNT> --container-name documents --auth-mode login -o table`.

Success criteria:
* Search index `cwyd-index` + Foundry IQ KB `cwyd-kb` exist; the default benefits PDFs are present in the `documents` container and enqueued for ingestion.

Dependencies:
* Step 3.2.

---

## Implementation Phase 4: Post-deploy validation (final validation)

<!-- parallelizable: false -->

Confirms the deployment is live, all three images rolled fresh, and the chat grounds on the seeded documents. This is the plan's final validation phase.

### Step 4.1: Confirm all three Container Apps pulled the new image

Files:
* (none)

Commands (read-only; resolve real names via `azd env get-values` first):
* `az containerapp revision list -g <RESOURCE_GROUP> -n <backend-ca-name> -o table`
* `az containerapp revision list -g <RESOURCE_GROUP> -n <frontend-ca-name> -o table`
* `az containerapp revision list -g <RESOURCE_GROUP> -n <function-ca-name> -o table`

Success criteria:
* Each app's newest revision is Active with TrafficWeight 100 and references the freshly-pushed ACR image tag (not the MCR placeholder).

Dependencies:
* Phase 3.

### Step 4.2: Backend health checks

Files:
* v2/src/backend health router (GET /api/health, GET /api/health/ready).

Commands:
* `curl https://<backend-fqdn>/api/health` — expect 200, body `status` in {pass, degraded}.
* `curl -i https://<backend-fqdn>/api/health/ready` — expect 200 (not 503).

Success criteria:
* `/api/health` returns `pass` (or `degraded` with a known non-fatal reason); `/api/health/ready` returns 200. Foundry + database + search sub-checks pass in cosmos mode.

Dependencies:
* Step 4.1.

### Step 4.3: Chat grounding smoke test

Files:
* v2/src/backend conversation router (POST /api/conversation).

Commands:
* POST `https://<backend-fqdn>/api/conversation` with `{"messages":[{"role":"user","content":"What is covered by the Northwind Health Plus plan?"}],"conversation_id":"smoke-001"}` and read the SSE stream.

Success criteria:
* The SSE stream emits `reasoning` → `answer` → at least one `citation` referencing a seeded benefits PDF (grounding proven — WI-03 grounding path validated). No `error` channel event.
* Per user memory (cleanup-before-next-step): delete the `smoke-001` conversation + any test artifacts after validation.

Dependencies:
* Step 3.3 (documents ingested). Ingestion may lag a few minutes after `azd up`; retry if the first query returns no citation.

### Step 4.4: Report + next steps

Files:
* (none — reporting only)

Success criteria:
* Deploy outcome recorded in the day's worklog (v2/docs/worklog/2026-07-01.md) per Hard Rule #19 (durable tracking).
* Any residual issues that exceed minor fixes are reported to the operator with recommended follow-up (no large-scale inline fixes).

Dependencies:
* Steps 4.1–4.3.

## Dependencies

* Live `az login` + `azd auth login` on the new tenant (Owner / User-Access-Administrator).
* Docker daemon (Phase 2.3 local builds).
* Confirmed `gpt-5.1` + `text-embedding-3-large` quota in `<REGION>`.

## Success Criteria

* `azd up` completes SUCCESS; all three Container Apps serve the freshly-built ACR images.
* `/api/health/ready` returns 200; a benefits question grounds with a citation.
* WI-07 resolved (frontend image builds); full vitest + build green.
