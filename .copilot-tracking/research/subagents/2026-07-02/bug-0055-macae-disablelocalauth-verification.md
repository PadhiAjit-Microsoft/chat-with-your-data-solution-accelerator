# BUG-0055 — MACAE `disableLocalAuth` verification (READ-ONLY research)

- **Date:** 2026-07-02
- **Repo:** microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator (branch `main`)
- **Status:** Complete
- **Method:** `github_text_search` (exact string `disableLocalAuth`) + `github_repo` semantic pulls of the surrounding Bicep blocks.

## Research questions

1. Does `disableLocalAuth: true` exist anywhere in MACAE's Bicep? (yes/no + count)
2. Is `disableLocalAuth` set on the Application Insights component (`Microsoft.Insights/components` / `avm/res/insights/component`)? If absent, say so explicitly (defaults to `false` = local auth enabled).
3. List each resource that has `disableLocalAuth: true`.

## Search scope confirmation

Exact-string search for `disableLocalAuth` across the repo returned **exactly 5 hits**:

- 4 in Bicep infra code (2 in `infra/main.bicep`, 2 in `infra/main_custom.bicep`).
- 1 in documentation prose only (`docs/TroubleShootingSteps.md`) — an App Configuration troubleshooting example, **not** an infra resource declaration.

No `.json` ARM template under `infra/` contains the string (infra is authored in Bicep; only `main.json.bak`-style artifacts exist in the *other* repo, not MACAE). The search is exhaustive.

## Every `disableLocalAuth` occurrence (infra Bicep)

| # | File | Approx. line | Module / AVM | Azure resource type | Resource (symbolic) | Value |
|---|------|-------------|--------------|--------------------|---------------------|-------|
| 1 | infra/main.bicep | ~880 | `br:mcr.microsoft.com/bicep/avm/res/cognitive-services/account:0.13.2` | `Microsoft.CognitiveServices/accounts` (kind `AIServices` = AI Foundry / AI Services / OpenAI) | `aiFoundryAiServices` | `true` |
| 2 | infra/main.bicep | ~1717 | `br/public:avm/res/search/search-service:0.12.0` | `Microsoft.Search/searchServices` (Azure AI Search) | `searchServiceUpdate` | `true` |
| 3 | infra/main_custom.bicep | ~880 | `br:mcr.microsoft.com/bicep/avm/res/cognitive-services/account:0.13.2` | `Microsoft.CognitiveServices/accounts` (kind `AIServices`) | `aiFoundryAiServices` | `true` |
| 4 | infra/main_custom.bicep | ~1767 | `br/public:avm/res/search/search-service:0.12.0` | `Microsoft.Search/searchServices` (Azure AI Search) | `searchServiceUpdate` | `true` |

Permalinks (branch `main`):

- Occurrence 1: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main.bicep — block at L870–L896 (`aiFoundryAiServices`, `disableLocalAuth: true`).
- Occurrence 2: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main.bicep — block at L1708–L1719 (`searchServiceUpdate`, `disableLocalAuth: true`).
- Occurrence 3: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main_custom.bicep — block at L864–L888 (`aiFoundryAiServices`, `disableLocalAuth: true`).
- Occurrence 4: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/infra/main_custom.bicep — block at L1762–L1790 (`searchServiceUpdate`, `disableLocalAuth: true`).

### Supporting code excerpts

AI Services (both files, identical shape):

```bicep
module aiFoundryAiServices 'br:mcr.microsoft.com/bicep/avm/res/cognitive-services/account:0.13.2' = if (!useExistingAiFoundryAiProject) {
  ...
  params: {
    name: aiFoundryAiServicesResourceName
    ...
    kind: 'AIServices'
    disableLocalAuth: true
    allowProjectManagement: true
    ...
  }
}
```

Azure AI Search (both files, identical shape — note a *separate* raw `Microsoft.Search/searchServices@2025-05-01` resource is created first, then this AVM `searchServiceUpdate` module sets the properties incl. `disableLocalAuth`):

```bicep
module searchServiceUpdate 'br/public:avm/res/search/search-service:0.12.0' = {
  name: take('avm.res.search.update.${solutionSuffix}', 64)
  params: {
    name: searchServiceName
    location: location
    disableLocalAuth: true
    hostingMode: 'Default'
    managedIdentities: { systemAssigned: true }
    publicNetworkAccess: 'Enabled'
    ...
  }
}
```

## Application Insights — explicit finding

Application Insights is deployed via `br/public:avm/res/insights/component:0.7.1` in both files:

- infra/main.bicep — module `applicationInsights`, block at L361–L376.
- infra/main_custom.bicep — module `applicationInsights`, block at L359–L376.

Full params block (verbatim, both files identical):

```bicep
module applicationInsights 'br/public:avm/res/insights/component:0.7.1' = if (enableMonitoring) {
  name: take('avm.res.insights.component.${applicationInsightsResourceName}', 64)
  params: {
    name: applicationInsightsResourceName
    tags: tags
    location: location
    enableTelemetry: enableTelemetry
    retentionInDays: 365
    kind: 'web'
    disableIpMasking: false
    flowType: 'Bluefield'
    workspaceResourceId: enableMonitoring ? logAnalyticsWorkspaceResourceId : ''
  }
}
```

**`disableLocalAuth` is NOT present on the Application Insights component in either file.** Therefore it defaults to `false` (the AVM `avm/res/insights/component` default and the `Microsoft.Insights/components` platform default) — i.e. **local auth is ENABLED** on MACAE's App Insights. MACAE does NOT disable local (instrumentation-key / connection-string) auth on Application Insights. Consistent with this, the container app / web app read `applicationInsights.outputs.instrumentationKey` and `applicationInsights.outputs.connectionString` and pass them as `APPLICATIONINSIGHTS_INSTRUMENTATION_KEY` / `APPLICATIONINSIGHTS_CONNECTION_STRING` env vars — key-based ingestion, which requires local auth to remain enabled.

## Resources that do NOT set `disableLocalAuth` (checked)

- **Application Insights** (`avm/res/insights/component`) — absent → defaults `false`.
- **Cosmos DB** (`avm/res/document-db/database-account:0.19.0`) — no `disableLocalAuth`; uses `networkRestrictions` + `sqlRoleDefinitions` (RBAC data-plane) but the key-disable flag is not set in the shown params.
- **Storage account** (`avm/res/storage/storage-account`) — no `disableLocalAuth`; uses `allowBlobPublicAccess: false`, `networkAcls`, `publicNetworkAccess`.
- **Container App / Web App / Log Analytics / Managed Identity / VNet / Bastion / VM** — none set `disableLocalAuth`.

## Answers

1. **Yes.** `disableLocalAuth: true` appears **4 times** in MACAE Bicep (2× AI Services, 2× AI Search; split across `main.bicep` and `main_custom.bicep`, one of each per file).
2. **No — App Insights does NOT set `disableLocalAuth`.** It is absent from the `avm/res/insights/component:0.7.1` params in both `main.bicep` and `main_custom.bicep`, so it defaults to `false` (local auth = enabled).
3. Resources with `disableLocalAuth: true`:
   - `Microsoft.CognitiveServices/accounts` (kind `AIServices`, AVM `cognitive-services/account:0.13.2`) — in `main.bicep` and `main_custom.bicep`.
   - `Microsoft.Search/searchServices` (AVM `search/search-service:0.12.0`, module `searchServiceUpdate`) — in `main.bicep` and `main_custom.bicep`.

## Recommended next research (not needed for this question)

- [ ] (Optional) Confirm the exact resolved line numbers by reading the raw files at a pinned commit SHA if permalinks must be immutable (current links point at `main` HEAD).
- [ ] (Optional) Verify the AVM `insights/component:0.7.1` module's own default for `disableLocalAuth` in the AVM registry if a hard citation of the default is required (platform default is `false`).

## Clarifying questions

None — the questions are fully answered by exhaustive exact-string search.
