<!-- markdownlint-disable-file -->
# Subagent Research: Recent CWYD v2 Changes — Redeploy Inventory (2026-06-25)

Inventory of the latest CWYD v2 changes so the operator can decide which deployable units
(backend, frontend, function) need redeploy to Azure. Read-only research (git + file reads).

Repo root: c:\workstation\Microsoft\github\cwyd-pg — all v2 code under v2/.

## Research Topics / Questions

1. What are the working-tree (unstaged) changes and the recent commits on the current branch?
2. Which deployable unit does each changed file belong to (backend / frontend / function / infra / tests-docs)?
3. What did the 2026-06-24 and 2026-06-25 worklogs change and why (esp. BUG-0084 RAI fix, BUG-0082 DB-stop)?
4. Which bugs are currently open?
5. Did the flagged files actually change (main.bicep, main.parameters.json, azure.yaml, definitions.py, content_safety.py, frontend src files)?
6. Conclusion: which units need redeploy?

## Status

Complete.

## Git State (exact output)

### Current branch

```
feature/cwyd-v2-6626
```

(HEAD is at origin/feature/cwyd-v2-6626 — branch is pushed/up to date.)

### git status --short

```
?? .copilot-tracking/research/2026-06-25/cwyd-v2-postgres-redeploy-research.md
```

Working tree is **clean** except for one untracked tracking-doc. **No uncommitted
source/infra changes.** The "latest changes" therefore live entirely in recent
**commits**, not the working tree.

### git diff --stat

```
(empty — no unstaged tracked changes)
```

### git log --oneline -20

```
63907b7 (HEAD -> feature/cwyd-v2-6626, origin/feature/cwyd-v2-6626) Document cloud deploy validation
638594e Replace KB-MCP Bicep with post-provision seeder
dc112dd Add postdeploy sample-data uploader and infra fixes
2428a0b v2: App Service frontend + infra parity changes
257103b Merge remote-tracking branch 'origin/dev-v2' into feature/cwyd-v2-6626
d33cf7c Update 2026-06-24.md
1e16f07 Add BUG-0084: admin prompt RAI rejection
e817548 Update 2026-06-24.md
558c3c9 Repin agent-framework to core; add guard test
afb6071 Add function telemetry and fix Azure hooks/env
3f21ab5 Mark BUG-0076 fixed and add 2026-06-23 worklog
46bdc9b Add assistant-type presets and admin wiring
7a03c4f Feature/cwyd v2 6626 (#2267)
830ae4f docs: mark BUG-0069 and BUG-0075 fixed
02159a8 Fix frontend build/lint; wire AZURE_ENVIRONMENT
92ac2aa ParserKey enum; move upload validation to service
a214182 blob_event: classify and dispatch create/delete
6923e8b Add URL download-to-blob ingestion pipeline
14fcd2b Add header user avatar with initials
1dcf304 Delete document also removes source blob
```

### Commit dates (recent window)

```
63907b7 2026-06-25 Document cloud deploy validation
638594e 2026-06-25 Replace KB-MCP Bicep with post-provision seeder
dc112dd 2026-06-25 Add postdeploy sample-data uploader and infra fixes
2428a0b 2026-06-25 v2: App Service frontend + infra parity changes
257103b 2026-06-24 Merge remote-tracking branch 'origin/dev-v2' into feature/cwyd-v2-6626
d33cf7c 2026-06-24 Update 2026-06-24.md
1e16f07 2026-06-24 Add BUG-0084: admin prompt RAI rejection
e817548 2026-06-24 Update 2026-06-24.md
558c3c9 2026-06-24 Repin agent-framework to core; add guard test
afb6071 2026-06-24 Add function telemetry and fix Azure hooks/env
3f21ab5 2026-06-23 Mark BUG-0076 fixed and add 2026-06-23 worklog
46bdc9b 2026-06-23 Add assistant-type presets and admin wiring
7a03c4f 2026-06-22 Feature/cwyd v2 6626 (#2267)
830ae4f 2026-06-22 docs: mark BUG-0069 and BUG-0075 fixed
```

## Per-Unit Classification Table

Aggregated from `git diff --name-status 257103b HEAD` (the post-merge 06-25 commits)
plus the three pre-merge functional commits 46bdc9b / afb6071 / 558c3c9 (06-23..06-24).

| File | Unit | Commit(s) | Functional vs cosmetic |
|---|---|---|---|
| v2/src/backend/core/agents/definitions.py | BACKEND | 46bdc9b, 638594e | 46bdc9b = functional (assistant presets + PROMPT_REVIEW_AGENT wiring); 638594e = cosmetic (docstring "macae"→"reference architecture") |
| v2/src/backend/core/agents/assistant_presets.json | BACKEND | 46bdc9b (new) | functional (ADR 0030 presets) |
| v2/src/backend/core/agents/presets.py | BACKEND | 46bdc9b (new) | functional |
| v2/src/backend/core/types.py | BACKEND | 46bdc9b | functional (AssistantType enum) |
| v2/src/backend/core/tools/content_safety.py | BACKEND | 638594e | cosmetic (docstring scrub only) |
| v2/src/backend/models/admin.py | BACKEND | 46bdc9b | functional (assistant type + prompt fields) |
| v2/src/backend/routers/admin.py | BACKEND | 46bdc9b | functional (validate_prompt_with_rai wiring, BUG-0084) |
| v2/src/backend/services/admin.py | BACKEND | 46bdc9b | functional (validate_prompt_with_rai allow-list + PROMPT_REVIEW_AGENT, BUG-0084) |
| v2/src/backend/services/conversation.py | BACKEND | 46bdc9b | functional |
| v2/src/frontend/frontend_app.py | FRONTEND | 2428a0b | functional (App Service SPA host) |
| v2/src/frontend/src/App.tsx | FRONTEND | 2428a0b | functional (runtime config bootstrap) |
| v2/src/frontend/src/api/admin.tsx | FRONTEND | 46bdc9b, 2428a0b | functional (backendUrl base + assistant type) |
| v2/src/frontend/src/api/conversationHistory.tsx | FRONTEND | 2428a0b | functional |
| v2/src/frontend/src/api/runtimeConfig.tsx | FRONTEND | 2428a0b (new) | functional (runtime config client) |
| v2/src/frontend/src/api/speech.tsx | FRONTEND | 2428a0b | functional |
| v2/src/frontend/src/api/streamChat.tsx | FRONTEND | 2428a0b | functional |
| v2/src/frontend/src/models/admin.tsx | FRONTEND | 46bdc9b | functional (assistant type model) |
| v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx | FRONTEND | 46bdc9b, 558c3c9 | functional (assistant-type dropdown + BUG-0083 normalizer) |
| v2/src/frontend/src/components/CoralShell/* (4 files) | FRONTEND | 638594e | cosmetic (macae scrub + minor css) |
| v2/src/frontend/src/components/Header/* (6 files) | FRONTEND | 638594e | cosmetic (macae scrub; MultiAgentLogo/userIdentity) |
| v2/src/frontend/src/pages/chat/* (8 files) | FRONTEND | 638594e | cosmetic (macae scrub + minor css) |
| v2/src/frontend/src/theme/FluentThemeBridge.tsx, theme/tokens.css | FRONTEND | 638594e | cosmetic |
| v2/src/frontend/tsconfig.tsbuildinfo | FRONTEND | 2428a0b, 63907b7 | build artifact (not runtime) |
| v2/src/functions/host.json | FUNCTION | dc112dd | functional (messageEncoding=none durable back-port, BUG-0056) |
| v2/src/functions/core/telemetry.py | FUNCTION | afb6071 (new) | functional (App Insights wiring, BUG-0055) |
| v2/src/functions/function_app.py | FUNCTION | afb6071 | functional (telemetry call) |
| v2/pyproject.toml + v2/uv.lock | FUNCTION/BACKEND deps | 558c3c9 | functional (repin agent-framework-core, BUG-0080 — unblocks function build) |
| v2/infra/main.bicep | INFRA | 2428a0b, dc112dd, afb6071, 638594e | functional (App Service frontend parity, env wiring, KB-MCP module removal) |
| v2/infra/main.json | INFRA | 2428a0b, dc112dd, 638594e | functional (regenerated ARM) |
| v2/infra/main.parameters.json | INFRA | 2428a0b | functional (new params) |
| v2/infra/modules/virtualNetwork.bicep | INFRA | 638594e | cosmetic (comment scrub) |
| v2/infra/modules/ai-project-kb-mcp-connection.bicep | INFRA | dc112dd (add) → 638594e (delete) | net removed (replaced by post-provision seeder) |
| v2/azure.yaml | INFRA | 2428a0b, dc112dd, afb6071, 638594e | functional (frontend service, hooks: prepackage/postprovision/postdeploy) |
| v2/scripts/post_provision.py | INFRA (postprovision hook) | 638594e | functional (KB-MCP seeder, BUG-0025/0059 durable) |
| v2/scripts/package_frontend.py + .ps1/.sh | INFRA (prepackage hook) | 2428a0b (new) | functional (frontend packaging) |
| v2/scripts/upload_sample_data.py + .ps1/.sh | INFRA (postdeploy hook) | dc112dd (new) | functional (sample-data seed) |
| v2/.gitignore | repo config | 2428a0b | not deployed |
| v2/tests/** (many) | TESTS | all | not deployed |
| v2/docs/** (bugs.md, worklog/*, adr/*) | DOCS | all | not deployed |
| .copilot-tracking/** | tracking docs | all | not deployed |

## Flagged-Files Confirmation

All six flagged files/areas **did change** in the recent window:

* v2/infra/main.bicep — YES (2428a0b, dc112dd, afb6071, 638594e) — functional.
* v2/infra/main.parameters.json — YES (2428a0b) — functional (new parameters for App Service frontend).
* v2/azure.yaml — YES (2428a0b, dc112dd, afb6071, 638594e) — functional (frontend service + hooks).
* v2/src/backend/core/agents/definitions.py — YES; but the **latest** (638594e) change is **cosmetic** (docstring "macae"→"reference architecture"). The functional content (PROMPT_REVIEW_AGENT, assistant presets) landed earlier in 46bdc9b and BUG-0084 fix; all committed and present in HEAD.
* v2/src/backend/core/tools/content_safety.py — YES; latest (638594e) change is **cosmetic only** (docstring scrub). Confirmed by diff: only comment/docstring lines changed, no executable code.
* v2/src/frontend/src/** (App.tsx, Header/*, api/admin.tsx, api/streamChat.tsx, api/speech.tsx, pages/chat/*) — YES. api/* and App.tsx (2428a0b) are **functional**; Header/* and pages/chat/* (638594e) are **cosmetic** macae-scrub + minor css.

## Worklog Summary

### v2/docs/worklog/2026-06-24.md (key events)

* **BUG-0080 (root cause + deploy unblock):** the multi-day function-deploy block was NOT a Flex platform outage — it was a dependency-resolution conflict: `pyproject.toml` pinned the umbrella `agent-framework==1.7.0`, which pulls `agent-framework-hyperlight` → unresolvable on the Functions host's Python 3.11. **Fix:** repin to `agent-framework-core==1.7.0` (commit 558c3c9). Deployed the pre-built package with `--no-build`; `blob_event` now live (6 functions). Local dev (Python 3.14) never reproduced it.
* **BUG-0082 (backend DB-stop incident):** backend Container App crash-looped because `psql-<SUFFIX>` (Burstable) had auto-stopped after 7 idle days, and the FastAPI lifespan's first DB call has **no connect timeout** → hung forever → permanent crash-loop. Mitigated by `az postgres flexible-server start` + restart. **Status still open** — the durable robustness fix (bounded connect timeout / fail-fast) is pending.
* **BUG-0081 (frontend never deploys via azd):** `azure.yaml` declares frontend as `host: appservice` + `docker:` block, but the azd `appservice` host does **not** support `docker`, so `azd package frontend` zips code and the placeholder container image is never replaced. Mitigated manually (`docker build → push ACR → az webapp config container set`), reverted by the next `azd provision`. **Still open** (durable azd-native fix is a pending design decision).
* **BUG-0083 (admin Assistant-type dropdown empty):** stale cloud backend omitted `ai_assistant_type`; FE normalizer added; backend redeployed.
* **BUG-0084 (admin prompt RAI rejection):** `PATCH /api/admin/config` screened the system prompt (incl. the built-in default) through the **user-message** jailbreak classifier `RAI_AGENT`, so the default prompt failed its own gate (422). **Fixed (functional, committed, in HEAD):** (a) deterministic allow-list in `validate_prompt_with_rai` for vetted built-in bodies; (b) new `PROMPT_REVIEW_AGENT` TRUE/FALSE classifier calibrated for administrator-authored system prompts. Backend redeployed 06-24 (cosmosdb env). Files: services/admin.py, routers/admin.py, definitions.py, models/admin.py, content_safety.py.
* **Cloud bring-up:** backend OK, function OK, frontend OK (manual container path).

### v2/docs/worklog/2026-06-25.md (key events)

* **KB-MCP Bicep → post-provision seeder:** deleted `ai-project-kb-mcp-connection.bicep`; added idempotent `_ensure_kb_mcp_connection` to scripts/post_provision.py; rewired `AZURE_AI_SEARCH_CONNECTION_NAME` to the seeded `cwyd-kb-mcp`; added `AZURE_AI_PROJECT_RESOURCE_ID` output (commit 638594e).
* **macae scrub:** every "macae" reference removed from shipped artifacts — backend (2 files = definitions.py + content_safety.py, **cosmetic docstrings**), frontend (20 files, **cosmetic**), tests (5), infra comments, .env, bugs.md attribution.
* **Deploy validation — cloud `azd up` (cosmosdb env):** end-to-end **green** — provision succeeded (after clearing one RBAC collision), KB-MCP seeder validated live, **backend Done, frontend Done, function Done**, sample-data seed ran. So the **cosmosdb env is at HEAD as of 06-25**.
* **Gotcha 1 — RBAC back-port collision:** `RoleAssignmentExists` on first redeploy when a previously-manual grant is back-ported to Bicep; delete the manual assignment, re-run `azd up`.
* **Gotcha 2 — function deploy + seed blocked by private-only storage:** the package/documents storage account is `publicNetworkAccess=Disabled`; `azd deploy function` uploads the zip from the workstation over the public internet → `403`. Workaround: temporarily open storage (`--public-network-access Enabled --default-action Allow`), deploy/seed, re-lock. Also: the postdeploy sample-data hook's `input()` hits EOF under azd (no TTY) — run non-interactively with `AZURE_ENV_SAMPLE_DATA=all`.

## Open Bugs (status open / in-progress)

From v2/docs/bugs.md registry (grep on open/in-progress):

| ID | Area | Severity | Status | One-line title |
|---|---|---|---|---|
| BUG-0054 | infra | medium | open | `doc-processing-poison` holds 10 Event Grid `BlobCreated` envelopes; EG→queue wiring needs the blob_event translator (ADR 0028); cloud deploy of the fix deferred. |
| BUG-0055 | infra | medium | open | Application Insights (`appi-<SUFFIX>`) has received zero telemetry from the function host and backend Container App (OTel export unwired/misconfigured). |
| BUG-0058 | functions | medium | open | `azd deploy function` did not run the `prepackage` hook → shipped a stale `build-functions/` artifact; workaround = run `prepackage_function.py` first. |
| BUG-0077 | functions | low | open | Enhancement: auto-remove a document from the index when its blob is deleted (Event Grid `BlobDeleted`); avoids stale index/pgvector chunks on bulk delete. |
| BUG-0081 | infra | high | open | Frontend never deploys via `azd deploy frontend` (azd `appservice` host does not support `docker`); manual container path is reverted by next `azd provision`. |
| BUG-0082 | backend | medium | open | Backend crash-loops when its PostgreSQL is unreachable; FastAPI lifespan DB calls have no connect timeout → infinite restart instead of fail-fast/degrade. (Incident mitigated 06-24; durable fix still open.) |

## Conclusion — Units needing redeploy

The answer depends on **which azd environment** is the target. The working tree is clean and
HEAD is pushed, so "redeploy" = "bring the target cloud env up to HEAD".

### If the target is the cosmosdb env

**None.** The 2026-06-25 worklog records a full `azd up` (cosmosdb) at HEAD with backend,
frontend, and function all deployed green. That env is already current.

### If the target is the PostgreSQL / pgvector env (the operative scenario — see the companion postgres-redeploy research)

**Backend = YES. Frontend = YES. Function = YES. Plus an infra provision is required.**

The PostgreSQL env was last provisioned/deployed around 2026-06-17 (BUG-0060 / BUG-0061
postgresql `azd up`), which **predates every recent change**. Justification per unit:

* **BACKEND — redeploy.** Functional changes since 06-17: assistant-type presets / ADR 0030 (46bdc9b: presets.py, assistant_presets.json, types.py, models/admin.py, services/conversation.py), the BUG-0084 RAI calibration (`PROMPT_REVIEW_AGENT` + `validate_prompt_with_rai` allow-list in services/admin.py + routers/admin.py — confirmed committed in HEAD), plus BUG-0076 / BUG-0083 fixes. The 06-25 docstring scrub is cosmetic but rides along. On pgvector the `agent_framework` orchestrator grounds app-side, so the backend is the only chat path — these changes matter.
* **FRONTEND — redeploy.** Functional changes (2428a0b): App Service SPA host (frontend_app.py), runtime-config bootstrap (App.tsx + new api/runtimeConfig.tsx), and `backendUrl` base + user-identity header wiring across api/admin.tsx, api/streamChat.tsx, api/speech.tsx, api/conversationHistory.tsx; plus the assistant-type dropdown + BUG-0083 normalizer (Configuration.tsx). **Caveat (BUG-0081):** `azd deploy frontend` does not actually replace the App Service container image — the manual `docker build → push ACR → az webapp config container set` path is required until BUG-0081's durable fix lands.
* **FUNCTION — redeploy.** Functional changes: host.json `messageEncoding=none` durable back-port (dc112dd, BUG-0056), App Insights telemetry wiring (afb6071: core/telemetry.py + function_app.py, BUG-0055), and the critical `agent-framework-core` repin (558c3c9, BUG-0080) that unblocks the build. **Caveats:** run the `prepackage` hook manually first (BUG-0058) and temporarily open the private-only storage account (06-25 Gotcha 2) before `azd deploy function`.
* **INFRA — provision required, not just `azd deploy`.** main.bicep, main.parameters.json, azure.yaml, virtualNetwork.bicep, main.json all changed, the KB-MCP Bicep module was removed in favor of the post_provision.py seeder, and new prepackage/postprovision/postdeploy hooks + scripts were added. A bare `azd deploy <service>` will not apply these — an `azd provision` (or `azd up`) is needed to pick up the parameter/Bicep/hook changes. **Note:** on pgvector mode the KB-MCP seeder no-ops (cosmosdb-only branch), and the postgresql `azd up` path has its own history (BUG-0060 ARM index fix, BUG-0061 Event Grid MI grant) that should be re-verified.

### Net recommendation

For the PostgreSQL env: **redeploy all three units (backend, frontend, function) and run an
`azd provision`/`azd up` for the infra changes**, observing the open caveats — BUG-0081
(manual frontend container path), BUG-0058 (run prepackage first), 06-25 Gotcha 2 (open
private storage for function deploy + seed), BUG-0082 (ensure `psql-<SUFFIX>` is started
before the backend boots), and the RBAC back-port collision (06-25 Gotcha 1).

## Recommended Next Research (not completed here)

- [ ] Confirm the actual last-deployed state of the **PostgreSQL** azd env (image tags / revisions in the cloud) rather than inferring from worklogs — requires reading `v2/.azure/<env>/.env` and/or live `az`/`azd` queries (out of scope for this read-only inventory).
- [ ] Determine whether the operator wants a minimal `azd deploy <service>` per-unit redeploy or a full `azd up` (the infra/hook changes argue for `azd up`).
- [ ] Verify the postgresql `azd up` path still provisions cleanly post-06-25 infra changes (BUG-0060 / BUG-0061 regression check).

## Clarifying Questions

1. **Which azd environment is the redeploy target — the cosmosdb env (already at HEAD) or the PostgreSQL/pgvector env (stale since ~06-17)?** The companion file `.copilot-tracking/research/2026-06-25/cwyd-v2-postgres-redeploy-research.md` implies PostgreSQL; confirming changes the answer from "nothing" to "all three units + provision".
2. Do you want a per-service `azd deploy` runbook, or a full `azd up` (the infra/parameter/hook changes mean a bare per-service deploy will miss the Bicep + hook updates)?

## References / Evidence

* git: `rev-parse --abbrev-ref HEAD`, `status --short`, `diff --stat`, `log --oneline -20`, `diff --name-status 257103b HEAD`, `show <sha> --name-status`, `show 638594e -- <backend files>` (diffs inspected).
* v2/docs/worklog/2026-06-24.md, v2/docs/worklog/2026-06-25.md.
* v2/docs/bugs.md (registry + open-bug grep).
* v2/src/backend/core/agents/definitions.py, v2/src/backend/services/admin.py, v2/src/backend/routers/admin.py (BUG-0084 fix presence confirmed).
* Companion: .copilot-tracking/research/2026-06-25/cwyd-v2-postgres-redeploy-research.md (skeleton — pending Phase 2).
