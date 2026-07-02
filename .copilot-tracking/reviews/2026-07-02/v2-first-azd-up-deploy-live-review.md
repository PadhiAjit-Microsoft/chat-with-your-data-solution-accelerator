<!-- markdownlint-disable-file -->
# Task Review: CWYD v2 — First `azd up` (Live Deploy Review)

**Review Date**: 2026-07-02
**Related Plan**: .copilot-tracking/plans/2026-07-01/v2-first-azd-up-deploy-plan.instructions.md
**Changes Log**: .copilot-tracking/changes/2026-07-01/v2-first-azd-up-deploy-changes.md
**Prior Review**: .copilot-tracking/reviews/2026-07-01/v2-first-azd-up-deploy-plan-review.md (Phase 1 code)
**Defect logged**: v2/docs/bugs.md → BUG-0093
**Worklog**: v2/docs/worklog/2026-07-02.md

## Scope

Live review of the executed **Phase 3 (`azd up`)** + **Phase 4 (post-deploy validation)** on the new tenant/subscription (SS1-Eng-Dev, eastus2, cosmosdb profile). The earlier review (2026-07-01) covered the Phase 1 code fix; this one covers the actual deployment.

## Severity Summary

| Severity | Count | Notes |
|----------|-------|-------|
| Critical | 1 | BUG-0093 — backend crashes on startup (Cosmos firewall); whole chat backend down |
| Major | 0 | — |
| Minor | 1 | `text-embedding-3-small` vs research-assumed `-large` (deployed fine; confirm intent) |

## Phase 3 — Provision + deploy: ✅ SUCCESS

`azd up` (run from the v2 sync terminal) provisioned + deployed cleanly:
- All infra: Log Analytics, ACR (`crcwyd2wrypa`), App Insights, Speech, Storage, Foundry + project, **gpt-5.1** + embedding model deployments, Content Safety, Container Apps Environment, Cosmos DB, Search, Foundry↔Search connection.
- All three Container Apps deployed with real ACR images (backend image tag `azd-deploy-1782949425`, targetPort 8000).
- Post-provision seeded `cwyd-index` (1536-dim) + `cwyd-kb` knowledge base + KB-MCP connection.
- Endpoints emitted (backend/frontend/function FQDNs).

Deployment note: azd must run with the **v2** directory as the shell cwd — a fresh async terminal defaults to the repo root and picks up the **v1** `azure.yaml` (services `adminweb`/`web`, `src/backend`), which fails; the azd `--cwd` flag did **not** redirect project discovery.

## Phase 4 — Post-deploy validation: ❌ FAIL (Critical)

| Check | Result |
|-------|--------|
| Backend Container App state | `provisioningState: Succeeded`, `runningStatus: Running`, revision Healthy, 1 replica, `Running: Activating` |
| `GET /api/health` | ❌ **Timeout, 0 bytes** (curl 60s + 90s both time out; HTTP 000) |
| Backend startup logs | uvicorn `Started server process` → `Waiting for application startup.` → **`CosmosHttpResponseError: (Forbidden) ... blocked by your Cosmos DB account firewall`** → `ERROR: Application startup failed. Exiting.` |
| Grounding smoke | Not reached (backend down) |
| Sample-data seed | Not completed (azd parked at the interactive seed menu; killed) |

### Critical finding — BUG-0093 (deployment-blocking)

The default public-profile deploy is **non-functional**: the backend reads a Cosmos DB container in its FastAPI startup lifespan and is `Forbidden` because **Cosmos DB + Storage came up with `publicNetworkAccess: Disabled`** while the profile is public (`enablePrivateNetworking=false`, no VNet / private endpoints). The public-egress Container Apps therefore can't reach them.

Evidence (live `az`):

| Service | publicNetworkAccess | Reachable |
|---|---|---|
| Cosmos DB | **Disabled** | ❌ (backend crash) |
| Storage | **Disabled** | ❌ (function ingestion/blobs) |
| Search | Enabled | ✅ |
| AI Services (Foundry) | Enabled | ✅ |

The bicep **intends** `publicNetworkAccess: enablePrivateNetworking ? 'Disabled' : 'Enabled'` on both Cosmos (`networkRestrictions.publicNetworkAccess`, main.bicep ~1314) and Storage (~1052) — and Search + Foundry prove the public profile is active — but the AVM modules `avm/res/document-db/database-account:0.19.0` and `avm/res/storage/storage-account:0.32.0` deploy `Disabled` regardless. Distinct from BUG-0062 (Storage `networkAcls.defaultAction`); this is the separate `publicNetworkAccess` switch. Security posture is preserved either way: `disableLocalAuth: true` → Cosmos is RBAC/Entra-only, so `publicNetworkAccess: Enabled` exposes no key-based surface.

## Follow-Up Recommendations

### Immediate unblock (live hotfix — reversible)
```
az cosmosdb update -g <RESOURCE_GROUP> -n cosno-<SUFFIX> --public-network-access ENABLED
az storage account update -g <RESOURCE_GROUP> -n st<SUFFIX> --public-network-access Enabled
```
Then restart the backend revision and re-check `/api/health`.

### Durable fix (BUG-0093, follow-up `/task-implement`)
Make Cosmos + Storage honor `publicNetworkAccess: Enabled` in the non-private-networking profile — verify the AVM module param path / bump or override the module, or re-assert in post-provision — then re-deploy and re-validate.

### Also
- Re-run the sample-data seed non-interactively (`--set default`) after the backend is up, then grounding smoke test.
- Confirm `text-embedding-3-small` is the intended embedding model (research assumed `-large`; index is 1536-dim which matches `-small`).

## Overall Status

🚫→✅ **Deployment recovered via live hotfix; durable fix still pending (BUG-0093 open).** Phase 3 (provision/deploy) was a clean success. Phase 4 initially failed on the Critical Cosmos/Storage public-access lockout, which was **unblocked live** (2026-07-02): Cosmos + Storage `publicNetworkAccess` set to `Enabled`, backend revision restarted → `/api/health` + `/api/health/ready` return `200 {"status":"pass"}` (foundry_iq / database / search all pass), frontend serves `200`. **Grounding validated end-to-end:** seeded 6 benefits PDFs (index reached 15 docs), and `POST /api/conversation` ("What does the Northwind Health Plus plan cover?") returned a grounded answer with a `[doc1]` citation to `Benefit_Options.pdf`. The **durable bicep/AVM fix (BUG-0093) is outstanding** — a fresh `azd up` would re-introduce the lockout until fixed. Also open: confirm `text-embedding-3-small` intent; delete the smoke-test conversation.
