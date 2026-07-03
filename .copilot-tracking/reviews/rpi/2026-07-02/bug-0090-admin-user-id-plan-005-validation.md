<!-- markdownlint-disable-file -->
# RPI Validation — BUG-0090 Phase 5: Documentation

- **Plan**: `.copilot-tracking/plans/2026-07-02/bug-0090-admin-user-id-plan.instructions.md`
- **Details**: `.copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md` (Phase 5 = Lines 264-319)
- **Changes log**: `.copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md`
- **Research**: `.copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md` ("Corrected BUG-0090 root cause" §88; "Documented security tradeoff" §261)
- **Phase**: 5 — Documentation (BUG-0090 correction, worklog, ADR, frontend no-op verification)
- **Validation date**: 2026-07-03
- **Status**: **Passed**

## Scope

Phase 5 has three steps (details Lines 264-319):

- **5.1** — Correct the `v2/docs/bugs.md` BUG-0090 registry row + append a `v2/docs/worklog/2026-07-02.md` entry; keep status `open` (flip at Phase 6).
- **5.2** — Author (or amend) an ADR for the auth-posture decision + revert path.
- **5.3** — Verify the frontend is already compliant (no code change); record a worklog no-op note.

Validation compared each plan/detail requirement against the actual docs: `v2/docs/bugs.md` (BUG-0090 row + adjacent BUG-0089/BUG-0091), `v2/docs/worklog/2026-07-02.md`, `v2/docs/adr/0031-backend-admin-auth-header-only-ingress-enforced.md`, and `v2/docs/adr/README.md`.

## Step-by-step comparison

### Step 5.1 — BUG-0090 registry row corrected + worklog appended — SATISFIED

Registry row `v2/docs/bugs.md` line 149 (physical, verified LEN=2513, single well-formed 7-column table row):

- **Root cause CORRECTED** ✓ — the row states: *"Root cause (corrected — the earlier 'no Easy Auth identity source feeds the backend' analysis was a mis-diagnosis of the retired split-host App Service topology): the 401 was the admin role gate working as designed. `backend.dependencies.requires_role("admin")` required the base64 `x-ms-client-principal` Easy Auth claims blob carrying an `admin` role, and fired only when `environment=production` and `require_admin_auth=true`; the SPA forwards only `x-ms-client-principal-id` (never the claims blob)..."* — matches research §88 exactly and explicitly repudiates the old "no Easy Auth identity source" mis-diagnosis. Evidence: `v2/docs/bugs.md` line 149.
- **Fix described** ✓ — *"collapsed the backend to a single `get_user_id` dependency that validates the `x-ms-client-principal-id` header is a GUID and otherwise falls back to the anonymous default GUID `00000000-0000-0000-0000-000000000000` (never 401); swapped every `/api/admin/*` route to it; deleted the entire Easy Auth role gate (`requires_role`, `REQUIRE_ADMIN_USER`, `AdminUserIdDep`, ...), the `require_admin_auth` setting, and the `AZURE_REQUIRE_ADMIN_AUTH` Bicep env var on `ca-backend-<SUFFIX>`. Identity enforcement, when enabled, is now an ingress/frontend concern ... matching MACAE."* — covers single-GUID `get_user_id`, gate/`require_admin_auth`/`AZURE_REQUIRE_ADMIN_AUTH` deletion, and ingress-level enforcement. Evidence: `v2/docs/bugs.md` line 149.
- **Status STILL `open`** ✓ — column 6 = `open`; Fixed-date column (col 3) empty; row tail: *"Status: open — awaiting Phase 6 live verification."* Not prematurely flipped to `fixed`. Matches details 5.1 ("leave `open` until then") and plan Step 6.3 (status flip is a Phase 6 step). Evidence: `v2/docs/bugs.md` line 149.
- **Security tradeoff + ADR link + revert path** ✓ — row tail links `adr/0031-backend-admin-auth-header-only-ingress-enforced.md` and names the revert path ("re-add the role gate + `require_admin_auth` + the Bicep env var"). Evidence: `v2/docs/bugs.md` line 149.
- **Adjacent rows intact + ordered** ✓ — line 148 `BUG-0089` (`fixed`), line 149 `BUG-0090` (`open`), line 150 `BUG-0091` (`fixed`); ascending order preserved, both neighbours full-content and untouched. Evidence: `v2/docs/bugs.md` lines 148-150.

Worklog `v2/docs/worklog/2026-07-02.md`:

- **Appended (not overwritten)** ✓ — the BUG-0090 "Done" bullet is the last entry in the `## Done` block (line 39) and a new `## Bugs` bullet (line 47); all prior day entries remain above it, including the two the plan calls out — `BUG-0055` (line 37) and `BUG-0081` (line 38) — plus `BUG-0093/0054/0088/0077/0058`. Evidence: `v2/docs/worklog/2026-07-02.md` lines 37-39, 46-47.
- **Change + corrected root cause + tradeoff** ✓ — line 39 records the corrected root cause, the Phases 1-4 delivered fix (single `get_user_id`, role-gate/`require_admin_auth`/`AZURE_REQUIRE_ADMIN_AUTH` deletion), the forgeable-header/ingress security tradeoff, and the ADR 0031 authoring; the `## Bugs` entry (line 47) repeats the corrected root cause and marks status "stays open — live verification pending Phase 6." Evidence: `v2/docs/worklog/2026-07-02.md` lines 39, 47.
- **Frontend no-op verification** ✓ — line 39 records: *"Frontend no-op verification (read-only, zero edits): confirmed the SPA already forwards `x-ms-client-principal-id` on every request (`api/auth.tsx` ...)."* Satisfies details Step 5.3's "record this as a no-op verification in the worklog." Evidence: `v2/docs/worklog/2026-07-02.md` line 39.

### Step 5.2 — ADR for the auth-posture decision + revert path — SATISFIED

New ADR `v2/docs/adr/0031-backend-admin-auth-header-only-ingress-enforced.md` (Status: Accepted, Date: 2026-07-02, Phase 5, Pillar: Configuration Layer over Stable Core):

- **Header-only / ingress-enforced posture** ✓ — Decision §§1-5: backend no longer enforces admin auth in app code; one `get_user_id` validates the header is a GUID and never raises; `user_id` is a trusted client-forgeable header; real auth is an ingress/frontend concern; frontend already compliant. Evidence: ADR 0031 "Decision" section.
- **Forgeable-header security tradeoff** ✓ — Consequences ("Security tradeoff (stated explicitly): `x-ms-client-principal-id` is a client-set, forgeable header ... admin routes — reads and writes — unless ingress-level authentication is enabled") plus a dedicated "Security tradeoff and mitigations" section (ingress auth + network restriction). Evidence: ADR 0031 "Consequences" + "Security tradeoff and mitigations".
- **Revert path** ✓ — "Revert path" section: re-add the role gate + claims decode/role extraction, re-add `require_admin_auth` gated on `environment=production` + `require_admin_auth=true`, re-add `AZURE_REQUIRE_ADMIN_AUTH` on `ca-backend-<SUFFIX>` + provision the identity source. Evidence: ADR 0031 "Revert path".
- **MACAE alignment** ✓ — Context + Consequences repeatedly anchor the posture to MACAE (`get_authenticated_user_details`, all-zeros fallback, never-raises, enforce-at-proxy). Evidence: ADR 0031 "Context" and "Consequences".
- **README index row** ✓ — `v2/docs/adr/README.md` line 45 carries the 0031 row with an accurate one-line summary and Phase 5 (`BUG-0090`) scope. Evidence: `v2/docs/adr/README.md` line 45.

### Step 5.3 — Frontend already compliant, no code change — SATISFIED

- No frontend file is listed as *modified for Phase 5* — the frontend no-op verification is recorded in the worklog (line 39, see 5.1 above). The one `auth.tsx` edit in the changes log is explicitly tagged `[Phase 6 / WI-05]` (doc-only docstring correction), not a Phase 5 change. Phase 5's "zero frontend edits" contract holds. Evidence: `v2/docs/worklog/2026-07-02.md` line 39; changes log "Modified" `auth.tsx` entry tagged `[Phase 6 / WI-05]`.

### Cross-cutting requirement #4 — No env-specific IDs + well-formed markdown — SATISFIED

- **Hard Rule #18** ✓ — the BUG-0090 registry row, ADR 0031, and the BUG-0090 worklog additions use only placeholder tokens (`ca-backend-<SUFFIX>`, `<AZD_ENV_NAME>`, `<RESOURCE_GROUP>`, etc.) and the all-zeros GUID sentinel `00000000-0000-0000-0000-000000000000` (an ADR-0019 carve-out). No real subscription/tenant/RG/UPN/suffix in any Phase 5 addition. Changes log records the env-ID gate (`v2/tests/shared/test_no_env_specific_content.py`) green (`shared 1039 passed`).
- **Markdown well-formed** ✓ — `bugs.md` carries `<!-- markdownlint-disable-file -->`; the BUG-0090 row is a single 7-column table line with no unescaped pipes; ADR 0031 is a conventionally structured ADR; the README 0031 entry is a valid table row.

## Findings

### Critical

- None.

### Major

- None.

### Minor

- **M1 — ADR 0031 "Follow-ups → Stale frontend docstring" note is itself now stale (cross-phase drift, informational).** ADR 0031's "Follow-ups" section states the `api/auth.tsx` header docstring *"still reads 'admin RBAC stays anchored on the backend's own server-injected Easy Auth claims' ... Correcting it is a frontend edit deliberately out of scope for the BUG-0090 documentation step; it is noted here as a follow-up."* The docstring was subsequently corrected in Phase 6 (changes log tags the `auth.tsx` edit `[Phase 6 / WI-05]`; the live docstring at `v2/src/frontend/src/api/auth.tsx` lines 30-32 now reads "backend validates only that it is a GUID ... at the ingress/proxy (Easy Auth injecting/overwriting this header)" — the old RBAC line is gone). The ADR note was accurate when authored in Phase 5, so this is not a Phase 5 defect; it is a stale forward-reference introduced by later Phase 6 work. Optional cleanup: strike or amend the ADR "Stale frontend docstring" follow-up to mark it resolved. Evidence: ADR 0031 "Follow-ups"; `v2/src/frontend/src/api/auth.tsx` lines 30-32; changes log `auth.tsx` `[Phase 6 / WI-05]` entry.

## Coverage assessment

Phase 5 is **fully covered**. All three steps (5.1 bug row + worklog, 5.2 ADR + README, 5.3 frontend no-op verification) are implemented and match both the details (Lines 264-319) and the research through-line ("Corrected BUG-0090 root cause" §88, "Documented security tradeoff" §261). The correct-but-still-`open` status discipline (flip deferred to Phase 6.3) is honored across all three artifacts. No plan item is missing; no requirement is contradicted. The single finding is a Minor, non-blocking cross-phase drift in an ADR follow-up note caused by later Phase 6 work.

## Notes on validation method

- The BUG-0090 row exceeds the 2000-char display truncation; its true content (LEN=2513) and tail were verified by an absolute-path read. An earlier relative-path terminal read produced a spurious "awaiting auth-architecture decision" tail from a wrong CWD; a targeted grep for that phrase across `v2/docs/bugs.md` returned zero matches, confirming no stale content exists in the actual row.

## Recommended next validations (not performed this session)

- [ ] Phase 6 (Validation) — confirm the recorded test tallies (backend 2176 passed / 1 skipped; infra 39; shared 1039), `az bicep build` exit 0, and pyright/get_errors clean claims against the actual suites.
- [ ] Phase 1-4 through-lines — validate `get_user_id` rewrite + admin route swap + role-gate deletion + Bicep `AZURE_REQUIRE_ADMIN_AUTH` removal against their changes-log entries.
- [ ] Optional Phase 5 cleanup — resolve M1 (ADR 0031 follow-up note) once the auth.tsx correction is acknowledged as landed.
- [ ] Step 6.3 pending — verify the BUG-0090 status flips to `fixed` (with a Fixed date) only after the live `GET /api/admin/status` → 200 check.

## Clarifying questions

1. Should the ADR 0031 "Follow-ups → Stale frontend docstring" note (M1) be amended now to mark the `auth.tsx` correction resolved, or intentionally left as the Phase-5-accurate record with the resolution tracked only in the Phase 6 changes log?
