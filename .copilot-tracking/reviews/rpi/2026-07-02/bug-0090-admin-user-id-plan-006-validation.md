<!-- markdownlint-disable-file -->
# RPI Validation: BUG-0090 — Admin 401 & user_id header handling — Phase 6 (Validation)

**Plan**: .copilot-tracking/plans/2026-07-02/bug-0090-admin-user-id-plan.instructions.md
**Details**: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Phase 6 = Lines 320-354)
**Changes log**: .copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md
**Planning log**: .copilot-tracking/plans/logs/2026-07-02/bug-0090-admin-user-id-log.md
**Phase**: 6 — Validation
**Validation date**: 2026-07-03
**Method**: read / grep / get_errors only (no files modified, no test suite executed, no deploy performed)

## Status

**Passed (delivered scope)** — Phase 6 overall remains **Partial / open** by design.

All Phase-6 work that has been delivered (Steps 6.1, 6.2, plus the two opportunistic
deliverables WI-05 and OBS-01) validates clean. Step 6.3 (live `azd` deploy +
`/api/admin/status` 200 verification + flip BUG-0090 → `fixed`) and Step 6.4 (final
report) remain **unchecked and correctly deferred** pending explicit user go-ahead per
Hard Rule #10 — a gated deferral, **not** a defect or a skip.

## Coverage Assessment

| Phase-6 item | Plan state | Delivered | Verdict |
| --- | --- | --- | --- |
| Step 6.1 — run backend/infra/bicep/env-ID/shared gates | `[x]` | Reported green in changes log | Pass (see Minor-1 — counts are a QV re-run item) |
| Step 6.2 — fix minor validation issues (scoped) | `[x]` | pyright/get_errors clean claimed; auth.tsx get_errors re-verified clean | Pass |
| WI-05 (opportunistic) — auth.tsx stale docstring | folded into Phase 6 | Done, doc-only, get_errors clean | Pass |
| OBS-01 (opportunistic) — BUG-0055 env-ID scrub | resolved in Phase 6 | Line 6 → `<AZD_ENV_NAME>`; no leak remains | Pass |
| Step 6.3 — live deploy + flip to `fixed` | `[ ]` | NOT run (gated) | Correctly deferred |
| Step 6.4 — report blocking issues | `[ ]` | Changes-log Release Summary partially serves this | Correctly deferred |

## Findings (severity-ordered)

### Critical

None.

### Major

None.

### Minor

**Minor-1 — Changes-log test counts are a Quality-Validation re-run item, not independently re-verified here (informational).**
The changes log asserts `backend 2176 passed / 1 skipped; infra 39 passed; shared 1039 passed (env-ID gate green); az bicep build exit 0`
(.copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md line ~72). These
counts are internally consistent with the plan's Success Criteria ("backend + infra tests,
bicep build, env-ID + shared gates all green"). As an RPI validator (read/grep/get_errors
only) I did **not** re-execute the suite — re-running `pytest v2/tests/backend`,
`pytest v2/tests/infra`, the env-ID gate, and `az bicep build` is a **Quality-Validation
step**, recommended below, not performed in this session. The one live spot-check within
RPI scope — `get_errors` on the single touched frontend file — is clean.

**Minor-2 — WI-05 (auth.tsx) has no explicit Phase-6 checklist step; traceability lives only in the changes log + planning log (informational).**
The auth.tsx docstring fix is recorded in the changes-log "Modified" section tagged
`[Phase 6 / WI-05]` and in the planning-log WI-05 row, but Implementation Phase 6 of the
plan checklist (Steps 6.1-6.4) contains no dedicated step for it. This is consistent with
WI-05's own note ("may be folded into Phase 6 cleanup or a separate frontend turn"), so it
is acceptable — but a reader following the checklist alone would not see the frontend edit.
No corrective action required; noted for traceability completeness.

## Evidence

### Claim 1 — auth.tsx `PRINCIPAL_ID_HEADER` docstring rewritten (doc-only; get_errors clean) — CONFIRMED

* [v2/src/frontend/src/api/auth.tsx](../../../../v2/src/frontend/src/api/auth.tsx#L27-L35) — the `PRINCIPAL_ID_HEADER` JSDoc now reads: "A browser-set value is forgeable and is **not** a trust boundary — the backend validates only that it is a GUID and otherwise treats it as the shared default partition. Real authentication, when enabled, is enforced at the ingress/proxy (Easy Auth injecting/overwriting this header), never in backend application code."
* The stale line "admin RBAC stays anchored on the backend's own server-injected Easy Auth claims" is **absent** — replaced by the GUID-validate + ingress-level-auth narrative required by WI-05.
* [v2/src/frontend/src/api/auth.tsx](../../../../v2/src/frontend/src/api/auth.tsx#L36) — the constant value `"x-ms-client-principal-id"` is **unchanged**; only the docstring changed → confirms "doc-only, no logic change."
* `get_errors` on auth.tsx → **No errors found.**

### Claim 2 — BUG-0055 research doc line ~6 uses the `<AZD_ENV_NAME>` placeholder (Hard Rule #18) — CONFIRMED

* [.copilot-tracking/research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md](../../../research/2026-07-02/bug-0055-appinsights-zero-telemetry-research.md#L6) line 6: `` `<AZD_ENV_NAME>` deployment, and determine the smallest correct fix. ``
* grep `AZD_ENV_NAME|AZURE_ENV_NAME` across the doc → **1 match, the placeholder only**. The leaked real env-name literal (formerly the `AZURE_ENV_NAME` value) no longer appears anywhere in the file. Hard Rule #18 satisfied.

### Claim 3 — BUG-0090 left `open`; live deploy DEFERRED (not skipped); no deploy executed — CONFIRMED

* [v2/docs/bugs.md](../../../bugs.md#L149) — BUG-0090 row: `| BUG-0090 | 2026-06-25 |  | infra | high | open |` (empty resolved-date column, status `open`). Row body: "**Fix (delivered — Phases 1-4; live verification pending Phase 6):** …".
* [v2/docs/worklog/2026-07-02.md](../../../worklog/2026-07-02.md#L39) — "(Phase 5 documentation; **live verification pending Phase 6**)"; and the Bugs-section entry: "**Status stays open — live verification pending Phase 6.**"
* [.copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md](../../../changes/2026-07-02/bug-0090-admin-user-id-changes.md#L73) — "**Deployment notes:** The live deploy (`azd provision` / `azd deploy backend` …) + `GET /api/admin/status` 200 verification is the only remaining step and needs explicit go-ahead. … BUG-0090 stays `open` until live-verified, then flips to `fixed`."
* Plan checklist [Step 6.3](../../../plans/2026-07-02/bug-0090-admin-user-id-plan.instructions.md) is `[ ]` unchecked, labeled "(gated on user go-ahead per Hard Rule #10)".
* **No deploy evidence for BUG-0090:** grep `azd provision|azd deploy|azd up` across the BUG-0090 artifacts returns **only** the deferred-instruction strings (changes.md:73, details.md:339). The worklog `azd deploy function` (line 34) and `azd provision` (line 37) entries belong to **BUG-0088** and **BUG-0055/0081** respectively — different bugs on the same working day, not a BUG-0090 deploy. Terminal history in context shows only `az login` (no `azd`). → The Step-6.3 deploy was **not** executed.

### Claim 4 — Changes-log validation claims internally consistent with plan Success Criteria — CONFIRMED (with Minor-1 note)

* Plan Success Criteria: "backend + infra tests, bicep build, env-ID + shared gates all green"; "`pyright --strict` clean on touched backend files."
* Changes-log Release Summary: "backend 2176 passed / 1 skipped; infra 39 passed; shared 1039 passed (env-ID gate green); `az bicep build` exit 0; `pyright`/`get_errors` clean on all touched files; grep-clean for every deleted symbol." → Internally consistent with the criteria.
* RPI-scope live spot-check: `get_errors` on auth.tsx clean. Full-suite re-execution deferred to Quality-Validation (Minor-1).

### Claim 5 — Follow-on items WI-04..WI-07 recorded — CONFIRMED

All four are present in the planning-log "Suggested Follow-On Work" section
([.copilot-tracking/plans/logs/2026-07-02/bug-0090-admin-user-id-log.md](../../../plans/logs/2026-07-02/bug-0090-admin-user-id-log.md)):

* **WI-04** — scrub pre-existing `#35b`/`#35c`/`#35e`/`#35c-4` task tokens from the per-route docstrings in admin.py (Hard Rule #16 debt not introduced here; not caught by the current gate so it does not block Phase 6). ✓ recorded.
* **WI-05** — correct the stale auth.tsx:27 docstring. ✓ recorded **and delivered** this phase (see Claim 1; changes-log `[Phase 6 / WI-05]`).
* **WI-06** — back-fill the missing `0030-assistant-type-prompt-presets.md` row in adr/README.md (pre-existing BUG-0076 index omission, ADR 0030 index gap). ✓ recorded.
* **WI-07** — reclassify the BUG-0090 registry `Area` from `infra` to `backend`. ✓ recorded; correctly left undone — bugs.md:149 still shows `Area=infra`, consistent with WI-07 being an optional follow-on, not part of this change.

## Recommended Next Validations (not performed this session)

- [ ] **Quality-Validation re-run (Minor-1):** re-execute `pytest v2/tests/backend -q`, `pytest v2/tests/infra -q`, `pytest v2/tests -k no_env_specific -q`, and `az bicep build v2/infra/main.bicep` to independently confirm the changes-log counts (backend 2176/1skip, infra 39, shared 1039, bicep exit 0).
- [ ] **Post-go-ahead deploy validation (Step 6.3):** after the user authorizes the shared-infra deploy, verify `GET /api/admin/status` returns 200 for both a GUID `x-ms-client-principal-id` header and the default-GUID header, then confirm BUG-0090 is flipped `open` → `fixed` with the resolved date filled in bugs.md and a worklog verification note.
- [ ] **`v2/infra/main.json` regeneration (DD-09):** confirm the untracked compiled ARM artifact drops the stale `AZURE_REQUIRE_ADMIN_AUTH` literal on the next `azd` build (untracked/gitignored → no leak, but worth confirming at deploy time).

## Clarifying Questions

1. None blocking. The only open item is procedural: do you want to authorize the Step-6.3 live deploy now (which would move Phase 6 from Partial to complete and flip BUG-0090 → `fixed`), or keep it deferred?
