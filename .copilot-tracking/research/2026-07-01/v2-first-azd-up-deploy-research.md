<!-- markdownlint-disable-file -->
# Research: CWYD v2 — First `azd up` Deploy Path (Primary Synthesis)

Status: Complete
Date: 2026-07-01
Scope: Planning-grade synthesis for the "full path to first `azd up`" plan (scope A). Consolidates two subagent investigations. READ-ONLY.

## Source subagent documents

* `.copilot-tracking/research/subagents/2026-07-01/v2-wi07-ts6133-fix-research.md` — WI-07 frontend build blocker.
* `.copilot-tracking/research/subagents/2026-07-01/v2-deploy-path-research.md` — azd deploy flow, grounding, validation, quota, gates.

Prior context (containerize + model-cleanup implementation, all complete + green):
* `.copilot-tracking/changes/2026-07-01/v2-containerize-services-and-model-cleanup-changes.md`
* `.copilot-tracking/plans/logs/2026-07-01/v2-containerize-services-and-model-cleanup-log.md` (WI-01…WI-09)

Starting state (from the prior plan): implementation complete — bicep build 0 errors, pyright strict 0/0/0, pytest 2686 passed (1 pre-existing fail: silent-except gate), vitest 594 passed (1 pre-existing fail: WI-07 audit-footer). No `azd up` has been run.

---

## 1. WI-07 — the one hard code blocker (frontend prod image build)

`Dockerfile.frontend --target prod` runs `npm run build` (`tsc -b && vite build`). `tsconfig.json:8` sets `noUnusedLocals: true`, so the unused `formatActor` helper at `Configuration.tsx:628` raises `TS6133` and fails the build → the frontend container image cannot build.

### Fix (recommended): UNCOMMENT the audit render

* Edit site: `v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx:1040-1045` — a 6-line JSX comment block (`{/* ... */}`) that renders the "Updated by" line via `formatActor(state.lastRuntime.updated_by)`. Delete the leading `{/* ` (line 1040) and trailing ` */}` (line 1045).
* Effect: `formatActor` (line 628) becomes used → `TS6133` clears. No other change needed.
* Data-safety: the render reads `state.lastRuntime.updated_by`; `RuntimeConfig.updated_by` already exists as a non-null `string` at `v2/src/frontend/src/models/admin.tsx:278`. No model, import, or CSS change.
* Coupled test (test-first is already satisfied — the test exists and currently fails): `v2/tests/frontend/pages/admin/Configuration/Configuration.test.tsx:505` — `"surfaces the audit footer with the runtime updated_at / updated_by metadata after save"`; assertion at line 525 `expect(footer).toHaveTextContent("admin-user-id")` (fixture `RUNTIME_FIXTURE.updated_by = "admin-user-id"`, line 78). Uncommenting makes it pass.
* #35d entanglement: NONE. #35d is marked cleared (`v2/docs/mvp_status.md:161`, `project_status.md`). The block only references committed, live symbols. "open #35d" in worklogs is stale ownership language.

### Verify commands

* From `v2/src/frontend/`: `npm run build` (or `npx tsc -b`) — expect no `TS6133`.
* From `v2/tests/frontend/`: `npx vitest run pages/admin/Configuration/Configuration.test.tsx -t "surfaces the audit footer"` — expect pass.

Alternative (rejected): remove `formatActor`. Fixes only the compiler error, forces weakening the existing test, discards a real audit-footer feature. Uncomment is smaller and fixes both failures.

Residual risk (low): the restored line surfaces a non-secret Entra object ID on the admin-only Configuration page — acceptable, admin-gated.

---

## 2. Deploy grounding — RESOLVED (WI-03 correct; 2026-06-25 survey stale)

A fresh `azd up` in cosmosdb mode **grounds out-of-the-box**:

* `v2/azure.yaml` has a project-level `postdeploy` hook → `scripts/upload-sample-data.ps1` / `.sh` → `v2/scripts/upload_sample_data.py`, which `upload_blob`s the default benefits PDFs into the `documents` container and enqueues ingestion.
* Sample docs live in the repo-root `data/` folder (`_curated_data_dir()` = `Path(__file__).parents[2] / "data"`). No `v2/data/` folder exists.
* `post_provision.py` (postprovision) seeds only the Search index schema + Foundry IQ KB (no documents) — the 2026-06-25 survey was right about *that script* but wrong that "no upload step exists anywhere."
* Non-interactive shell + no override → seeds the DEFAULT benefits set. Opt out with `AZURE_ENV_SAMPLE_DATA=none` or menu `0/skip`. Index/KB are schema-only + empty only between provision and deploy, or when the seed is skipped.

Implication for the plan: post-deploy grounding validation is expected to succeed on a default-benefits question (returns a citation), not just "no grounding."

---

## 3. `azd up` flow + required env

* `cd v2` → `azd auth login` → `azd env new <AZD_ENV_NAME>` (or select existing) → optionally `azd env set AZURE_ENV_DATABASE_TYPE cosmosdb`, `azd env set AZURE_ENV_AI_SERVICE_LOCATION eastus2` → `azd up`.
* Prompts when unset: subscription → `AZURE_LOCATION` (allowed: `australiaeast` / `eastus2` / `japaneast` / `uksouth`) → `databaseType` (default `cosmosdb`) → `azureAiServiceLocation` (default `eastus2`) → 4 WAF booleans (all default `false`).
* Required: `AZURE_LOCATION` (no default). Effectively-required: `AZURE_ENV_AI_SERVICE_LOCATION`. `AZURE_PRINCIPAL_ID` is auto-set by azd.
* azd version observed: `1.27.0` — satisfies the repo pin `>= 1.18.0 != 1.23.9` (WI-02 ✓). Produces unique per-build tags for `host: containerapp` services (default `azd-deploy-<ts>`), so every deploy rolls fresh code + a new revision.

---

## 4. Pre-deploy verification commands (placeholders only — never real IDs)

### Quota (WI-01) — two model deployments

| Deployment | Model | Version | SKU | Capacity |
|---|---|---|---|---|
| chat | `gpt-5.1` | `2025-11-13` | GlobalStandard | 150 |
| embedding | `text-embedding-3-large` | `1` | Standard | 100 |

```powershell
az cognitiveservices usage list --location <REGION> --query "[?contains(name.value, 'OpenAI.GlobalStandard.gpt-5.1')].{name:name.value,current:currentValue,limit:limit}" -o table
az cognitiveservices usage list --location <REGION> --query "[?contains(name.value, 'OpenAI.Standard.text-embedding-3-large')].{name:name.value,current:currentValue,limit:limit}" -o table
```

### Local docker builds (run from `v2/`, context `.`)

```powershell
docker build -f docker/Dockerfile.backend  -t cwyd-backend:local .
docker build -f docker/Dockerfile.frontend --target prod -t cwyd-frontend:local .
docker build -f docker/Dockerfile.functions -t cwyd-function:local .
```

Frontend build fails until WI-07 is fixed (Phase 1). `remoteBuild: true` means ACR builds the image regardless, but the same `tsc` step likely fails in ACR too — so WI-07 must precede any deploy.

### what-if preview — resourceGroup scope (NOT subscription)

`main.bicep` `targetScope = 'resourceGroup'` (`main.bicep:25`). Use `azd provision --preview` (preferred, uses azd's parameter binding) or `az deployment group what-if -g <RESOURCE_GROUP> -f v2/infra/main.bicep -p v2/infra/main.parameters.json`. Do NOT use `az deployment sub what-if`.

---

## 5. Post-deploy validation

* Confirm each container app pulled the new image: `az containerapp revision list -g <RESOURCE_GROUP> -n ca-<svc>-<SUFFIX> -o table` — newest revision Active, TrafficWeight 100. (Confirm actual resource names via `azd env get-values` — `cloud_deployment.md` predates the containerapp conversion.)
* Health: `GET /api/health` (always 200; body `status` = pass/degraded/fail) and `GET /api/health/ready` (503 on FAIL). Shallow checks: foundry (`AZURE_AI_PROJECT_ENDPOINT` + `AZURE_OPENAI_GPT_DEPLOYMENT`), database endpoint, search endpoint (skipped in pgvector).
* Chat smoke: `POST /api/conversation` with `{"messages":[{"role":"user","content":"What is covered by the Northwind Health Plus plan?"}],"conversation_id":"smoke-001"}` — SSE channels reasoning/tool/answer/citation/error. With the postdeploy seed, a default-benefits question grounds with a citation.

---

## 6. Non-blockers / out-of-scope for this deploy

* Silent-except gate: `test_no_silent_excepts` flags `v2/src/functions/core/search_resolution.py:~91` `except BaseException:` (re-raises, but the gate bans `except BaseException` unconditionally). It is a hard pytest/CI gate (red test) but `azd up`/`provision`/`deploy` never run pytest → does NOT block a deploy. Phase 6 work owned by a separate effort. Leave out of this plan.
* WI-06 / WI-08: unused `web` / `functions` subnets — private-networking profile only; out of the default public `azd up` path.
* WI-09: `v2/src/functions/pyproject.toml` retirement — confirm local `func host start` first; not a deploy blocker.

---

## 7. Deploy-readiness — ordered blockers

1. **WI-07 `TS6133`** (Phase 1) — the only hard code blocker; frontend image will not build until fixed.
2. **Quota (WI-01)** — verify `gpt-5.1` GlobalStandard + `text-embedding-3-large` Standard in the target region before provision (fresh tenant).
3. **Deployer role** — Owner / User-Access-Administrator for the role assignments in `main.bicep`.
4. Local docker builds + `azd provision --preview` — recommended pre-`azd up` gates.

Everything else (azd version, grounding, hooks) is verified working. Net: after WI-07 + quota confirmation, the repo is ready for `azd up`.

## Open items for implementation-time verification

* [ ] Confirm actual Container App resource names post-conversion via `azd env get-values` (runbook `cloud_deployment.md` is stale).
* [ ] Verify the frontend ACR remote build clears TS6133 after the Phase 1 fix (not just the local build).
* [ ] Confirm the deployed `ingestionTrigger` default (`event_grid` vs `direct_enqueue`) in the target env — affects whether the seed enqueues automatically.
