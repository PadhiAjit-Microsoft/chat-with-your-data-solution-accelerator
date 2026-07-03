<!-- markdownlint-disable-file -->
# RPI Validation — BUG-0055 App Insights local auth — Phase 2 (ADR-0018 amendment)

- **Validation date**: 2026-07-02
- **Plan**: `.copilot-tracking/plans/2026-07-02/bug-0055-appinsights-local-auth-plan.instructions.md`
- **Changes log**: `.copilot-tracking/changes/2026-07-02/bug-0055-appinsights-local-auth-changes.md`
- **Details**: `.copilot-tracking/details/2026-07-02/bug-0055-appinsights-local-auth-details.md`
- **Research**: `.copilot-tracking/research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md`
- **Phase under validation**: Phase 2 — "ADR-0018 amendment" (Step 2.1: Record the reversal as ADR-0018 Amendment 2)
- **Artifact verified**: `v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md`

## Phase Status: **VERIFIED**

Step 2.1 is fully implemented. The `## Amendment 2` section exists, is correctly placed, matches the Amendment 1 heading + prose style, covers all four required content elements (a)-(d) accurately, names both revert call sites and the sync credential, and contains no env-specific IDs. One **minor** accuracy nuance is noted below (characterization of reversed Alternative #3) — it does not block the phase.

## Structural Placement Check

| Element | Line | Result |
| --- | --- | --- |
| `## Amendment 1 (2026-06-23) — per-workload App Insights env-var names (BUG-0055)` | 74 | reference anchor |
| `## Amendment 2 (2026-07-02) — enable App Insights local auth to match MACAE (BUG-0055)` | **102** | ✅ present |
| `## References` | 148 | reference anchor |

- **Placement**: Amendment 2 (L102) is **after** Amendment 1 (L74) and **before** References (L148). ✅
- **Heading style match**: Amendment 2 uses the identical `## Amendment N (YYYY-MM-DD) — <descriptor> (BUG-0055)` pattern as Amendment 1, including the em-dash separator and trailing `(BUG-0055)` tag. ✅
- **Prose style match**: bolded lead-in sub-sections (`**What changed.**`, `**Why.**`, `**Tradeoff.**`, `**Revert path.**`) mirror Amendment 1's explanatory-paragraph tone. ✅

## Content Elements Checklist (a)-(d)

### (a) What changed — ✅ PRESENT (L114-118)

> "**What changed.** The App Insights component `disableLocalAuth` flips from `true` to `false` in [`v2/infra/main.bicep`]…"

> "The `Monitoring Metrics Publisher` role assignment on the App Insights component (introduced by Decision #2 above) is **retained** even though it is now unused, kept in place to preserve a clean revert path (see below). No application code changes."

Covers both required facts: (1) `disableLocalAuth: true` → `false` on the App Insights component; (2) `Monitoring Metrics Publisher` role retained but now unused. ✅

### (b) Why — ✅ PRESENT (L120-129)

> "**Why.** This matches MACAE (the Multi-Agent Custom Automation Engine Solution Accelerator), whose App Insights `avm/res/insights/component` **omits** `disableLocalAuth` — defaulting to `false` — and ingests telemetry with connection-string / instrumentation-key auth…"

> "CWYD's application code is **already** connection-string-only (`configure_azure_monitor(connection_string=...)`), so enabling local auth restores ingestion with **zero application-code change**. This is the smallest possible fix for BUG-0055…"

Covers: match MACAE's proven pattern; app already connection-string-only → zero app-code change; smallest possible fix. ✅ Accurate against research (MACAE omits `disableLocalAuth`, grants no publisher role) and the subagent MACAE-verification.

### (c) Tradeoff — ✅ PRESENT (L131-137)

> "**Tradeoff.** Instrumentation-key / connection-string ingestion is a **weaker auth bar** than Entra-token ingestion. Accepted because MACAE — a shipped Microsoft reference accelerator — uses exactly this posture."

> "This amendment therefore **reverses** the original ADR's Decision #2 (Entra-only ingestion via `disableLocalAuth: true`), its *Positive* consequence "**UAMI + Entra-only ingestion stays**", and rejected Alternative #3…"

Covers: ikey ingestion is a weaker bar than Entra-token; accepted because MACAE uses this posture; **explicitly names the reversed ADR elements** — Decision #2, the *Positive* consequence "UAMI + Entra-only ingestion stays", and Alternative #3. ✅ (Requirement 3 — "names which original ADR elements it reverses" — satisfied.)

### (d) Revert path — ✅ PRESENT (L139-146)

> "**Revert path.** To return to Entra-only ingestion, flip `disableLocalAuth` back to `true` and pass a **synchronous** `azure.identity.ManagedIdentityCredential(client_id=...)` to `configure_azure_monitor(connection_string=..., credential=...)` at **both** sites: the backend lifespan in [`v2/src/backend/app.py`]… and the functions worker in [`v2/src/functions/core/telemetry.py`]."

> "Because the `Monitoring Metrics Publisher` role assignment is retained, no RBAC change is required for the revert."

Covers: flip back to `true`; pass a **sync** `ManagedIdentityCredential`; **both** call sites named — backend `app.py` lifespan + functions `core/telemetry.py` worker; retained role makes the revert require no RBAC change (one-flip revert). ✅ (Requirement 4 satisfied.)

## Env-Specific ID Check (Hard Rule #18) — ✅ PASS

Scan of the Amendment 2 block (L102-146) found **no** subscription ID, tenant ID, resource-group name, azd env name, resource suffix, UAMI client/principal ID, or FQDN. Identifiers are generic: `disableLocalAuth`, `avm/res/insights/component`, `Monitoring Metrics Publisher`, `configure_azure_monitor`, `ManagedIdentityCredential(client_id=...)` (placeholder ellipsis, not a real value), and relative repo file paths. No CRITICAL leak. ✅

> Note: the built-in role GUID `3913510d-42f4-4e42-8a64-81b1edca285c` appears elsewhere in the ADR body/References but **not** in the Amendment 2 section; it is a built-in Azure role-definition GUID (Hard Rule #18 carve-out), so it is a non-finding regardless.

## Research Consistency Check (Requirement 7) — ✅ PASS

The amendment's **primary** narrative is the SELECTED match-MACAE approach (`disableLocalAuth: false`, connection-string ingestion, zero app-code change), consistent with the research doc's `## Outline` selected fix and `Complete Examples → Selected (match MACAE)`. The credential/token approach appears **only** in the "Revert path" sub-section and is explicitly labeled "the research doc's rejected alternative, preserved here for reference." The rejected approach is never presented as primary. ✅

## Phase-Scope Check (Requirement 6) — ✅ PASS (ADR-only)

Per Details Step 2.1, Phase 2's `Files:` list contains **only** `v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md`. The changes-log `Modified` entries for `v2/infra/main.bicep` and `v2/tests/infra/test_main_bicep.py` are attributed to **Phase 1** (Steps 1.1-1.3), not Phase 2. Phase 2 correctly modified the ADR file only. ✅

> Verification basis: plan phase decomposition + details `Files:` attribution + changes-log grouping. A working-tree diff reflects Phases 1+2 combined and cannot isolate Phase 2 by itself; the documented scope is the authority for phase attribution.

## Findings by Severity

### Critical — none

### Major — none

### Minor

- **M-1 (accuracy nuance, L136-137)** — The amendment glosses reversed **Alternative #3** as one "which had considered — and dismissed — a posture with local auth left enabled." The ADR's actual Alternative #3 (L69) is *"Default-on monitoring but skip the `Monitoring Metrics Publisher` role assignment"* — it kept `disableLocalAuth: true` and dropped the role, producing the silent-401 drop; it was not framed as "leave local auth enabled." Alternative #3 is the *closest* analog in the ADR's alternatives list (the ADR never enumerated an explicit "leave local auth on" option), and the plan/details Step 2.1 explicitly directed naming Alternative #3 as a reversed element, so the implementer followed the plan. The literal requirement (c) — "names which original ADR elements it reverses" — is satisfied. Recommendation (non-blocking, for a future doc-polish turn): soften the parenthetical to note Alternative #3 is the nearest-analog reversal rather than a direct "local auth left enabled" consideration.

### Info

- **I-1** — Amendment 2's opening ("Amendment 1 fixed the backend env-var name but both runtimes still emitted **zero** telemetry") is internally consistent with Amendment 1's closing statement that "the function half of BUG-0055 … remains open." Good cross-amendment continuity.
- **I-2** — The amendment's MACAE factual claims (component omits `disableLocalAuth`; grants no `Monitoring Metrics Publisher` role) match the research doc's External Research + the `bug-0055-macae-disablelocalauth-verification` subagent doc. Accurate.

## Coverage Assessment

Step 2.1 is the sole step in Phase 2. It is **fully covered**: heading present and correctly placed (L102), style-matched, all four content elements (a)-(d) present and accurate, revert path names both call sites + sync credential, reversed ADR elements explicitly named, no env-ID leak, and research-consistent. Coverage: **100%** of Phase 2 requirements, one minor accuracy nuance (M-1).

## Clarifying Questions

- None. All Phase 2 acceptance criteria were resolvable from the artifacts and the verified ADR file.

## Recommended Next Validations (not performed this session)

- [ ] Phase 1 (Steps 1.1-1.4) — Bicep `disableLocalAuth: false` single-line flip (App Insights slice only; siblings still `true`), the two infra tests, the stale-message prose refresh, and `az bicep build` clean.
- [ ] Phase 3 (Steps 3.1-3.3) — `azd provision`, live KQL telemetry verification, and BUG-0055 close-out in `bugs.md` + worklog. (Changes log states Phase 3 is intentionally not yet executed.)
- [ ] Phase 4 (Steps 4.1-4.3) — full infra + `test_no_env_specific_content.py` gate run.
