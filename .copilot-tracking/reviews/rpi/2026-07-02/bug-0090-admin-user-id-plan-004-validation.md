<!-- markdownlint-disable-file -->
# RPI Validation — BUG-0090 Phase 4: Bicep — remove `AZURE_REQUIRE_ADMIN_AUTH`, refresh `AZURE_ENVIRONMENT` comment

**Validation date:** 2026-07-03
**Phase:** 4 (of 6)
**Status:** Passed

## Inputs

* Plan: `.copilot-tracking/plans/2026-07-02/bug-0090-admin-user-id-plan.instructions.md`
* Details: `.copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md` (Phase 4 = lines 216-263)
* Changes log: `.copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md`
* Research: `.copilot-tracking/research/subagents/2026-07-02/bug-0090-bicep-env-and-environment-usage-research.md`
* Source validated: `v2/infra/main.bicep`, `v2/tests/infra/test_main_bicep.py`

## Coverage Assessment

Phase 4 is **fully implemented**. Both plan steps (4.1 bicep edit, 4.2 infra test) landed and are verified against the actual source. All five requested verification points pass. No Critical or Major deviations found; two Minor/informational notes recorded below (both expected, non-blocking, and already documented in the changes log).

| Plan step | Requirement | Status | Evidence |
| --- | --- | --- | --- |
| 4.1 | Delete backend `AZURE_REQUIRE_ADMIN_AUTH` entry + comment | Complete | Grep: zero matches in `main.bicep` |
| 4.1 | Keep backend `AZURE_ENVIRONMENT` entry, rewrite comment (no auth narrative) | Complete | [v2/infra/main.bicep](../../../../../v2/infra/main.bicep#L1795-L1797) |
| 4.1 | Functions `AZURE_ENVIRONMENT` + comment UNCHANGED | Complete | [v2/infra/main.bicep](../../../../../v2/infra/main.bicep#L2142-L2144) |
| 4.1 | No env-specific IDs (Hard Rule #18) | Complete | Only `'production'` literal + `subscription().tenantId` / module outputs |
| 4.2 | Test asserts `AZURE_REQUIRE_ADMIN_AUTH` absent + `AZURE_ENVIRONMENT` present | Complete | [v2/tests/infra/test_main_bicep.py](../../../../../v2/tests/infra/test_main_bicep.py#L176-L200) |
| 4.2 | `pytest tests/infra/test_main_bicep.py` passes | Complete | 39 passed in 0.04s |

## Verification Points

### 1. `AZURE_REQUIRE_ADMIN_AUTH` has zero matches in `v2/infra/main.bicep` — PASS

Grep over `v2/infra/**` returns a single hit, and it is in the compiled ARM artifact, not the Bicep source:

* `v2/infra/main.json:48372` — compiled ARM output carries the literal (`createObject('name', 'AZURE_REQUIRE_ADMIN_AUTH', 'value', 'false')`).
* Zero matches in `v2/infra/main.bicep`.

`main.json` is **untracked and gitignored** — confirmed:

* `git check-ignore v2/infra/main.json` → `v2/infra/main.json` (ignored).
* `git ls-files --error-unmatch v2/infra/main.json` → exit 1 (not tracked).

Per the task instruction, the `main.json` residue is **not blocking**. It regenerates on the next `azd`/`az bicep build` (changes log documents this as deferred to Phase 6). Since it is untracked it cannot leak into version control. Recorded as Minor note MN-01.

### 2. Backend `AZURE_ENVIRONMENT` entry retained with a rewritten, auth-free comment — PASS

At [v2/infra/main.bicep](../../../../../v2/infra/main.bicep#L1795-L1797):

```bicep
            // Runtime mode: sets AppSettings.environment, surfaced by
            // GET /api/admin/status. It no longer governs any auth behavior.
            { name: 'AZURE_ENVIRONMENT', value: 'production' }
```

* Entry is **retained** (`{ name: 'AZURE_ENVIRONMENT', value: 'production' }`).
* Comment is rewritten to an accurate present-tense note matching the plan's specified wording ("sets `AppSettings.environment`, surfaced by `GET /api/admin/status`; no longer governs auth").
* The stale 9-line narrative from the research snippet (`local-dev identity bypass`, `folds an anonymous caller into the synthetic 'local-dev' partition`, `admin auth WALL ... controlled separately by AZURE_REQUIRE_ADMIN_AUTH below`) is **fully removed** — no `require_admin_auth`, no local-dev-bypass, no auth-wall language remains.

Hard Rule #16 (no process narrative) applies only to `v2/src/**`, so it does not govern this Bicep comment; nonetheless the rewritten comment is clean and descriptive. The phrase "no longer governs" is a mild backward-reference but matches the plan's own prescribed text and describes present state accurately — no finding.

### 3. Functions `AZURE_ENVIRONMENT` entry + comment UNCHANGED — PASS

At [v2/infra/main.bicep](../../../../../v2/infra/main.bicep#L2142-L2144):

```bicep
              // Runtime mode (AppSettings.environment) -- pin 'production' so the
              // deployed config reports production, parity with the backend.
              { name: 'AZURE_ENVIRONMENT', value: 'production' }
```

Byte-for-byte identical to the functions snippet captured in the research doc (research §"Exact Bicep snippets", functions block). No edit was made to the functions env block. Confirmed UNCHANGED.

### 4. Infra test asserts absence + presence, scoped to backend slice — PASS

At [v2/tests/infra/test_main_bicep.py](../../../../../v2/tests/infra/test_main_bicep.py#L176-L200), `test_backend_aca_env_drops_require_admin_auth_keeps_environment`:

```python
    assert "AZURE_REQUIRE_ADMIN_AUTH" not in backend_aca_slice, (...)
    assert "'AZURE_ENVIRONMENT'" in backend_aca_slice, (...)
```

The assertion is **meaningfully scoped**: `backend_aca_slice` (fixture at [v2/tests/infra/test_main_bicep.py](../../../../../v2/tests/infra/test_main_bicep.py#L118-L127)) slices the source between `module backendContainerApp ` and `module frontendContainerApp `, so the absence check targets the backend env block only — not a whole-file grep that could be satisfied incidentally. Both assertions carry descriptive failure messages.

Test run: `python -m pytest tests/infra/test_main_bicep.py -q` → **39 passed in 0.04s** (green against the actual `main.bicep`).

### 5. No env-specific IDs introduced (Hard Rule #18) — PASS

The Phase 4 edits introduce only:

* the string literal `'production'`,
* generic comment prose referencing `AppSettings.environment` and `GET /api/admin/status`,
* env-var name literals in the test file.

No subscription ID, tenant ID, UAMI client/principal ID, resource-group name, azd env name, or resource suffix appears in either touched file's changed region. Existing dynamic references (`subscription().tenantId`, `userAssignedIdentity.outputs.clientId`, module outputs) are template expressions, not literal IDs. The infra env-ID gate is reported green in the changes log (shared 1039 passed). Clean.

## Findings

No Critical findings.
No Major findings.

### Minor / Informational

* **MN-01 (Informational).** `v2/infra/main.json:48372` (compiled ARM artifact) still carries the `AZURE_REQUIRE_ADMIN_AUTH` literal. Untracked + gitignored (verified), regenerates on next build. Explicitly non-blocking per task instruction and already documented in the changes log ("Additional or Deviating Changes", Phase 4). No action required for Phase 4; regeneration lands with the Phase 6 live deploy.
* **MN-02 (Informational).** Line-number drift between the plan/details and the current source: details reference backend `AZURE_ENVIRONMENT` at `main.bicep:1805` and functions at `:2160`; the delivered file has them at `:1797` and `:2144`. This is the expected consequence of deleting the 8-line `AZURE_REQUIRE_ADMIN_AUTH` comment+entry and shortening the `AZURE_ENVIRONMENT` comment (9 lines → 2). The plan/details line refs describe the pre-edit state; the changes log documents the removal correctly. Cosmetic only — no defect.

## File Evidence Summary

| File | Referenced in changes log | Verified present | Notes |
| --- | --- | --- | --- |
| `v2/infra/main.bicep` | Yes ([Phase 4]) | Yes | `AZURE_REQUIRE_ADMIN_AUTH` removed; backend `AZURE_ENVIRONMENT` kept + re-commented (L1795-1797); functions block untouched (L2142-2144) |
| `v2/tests/infra/test_main_bicep.py` | Yes ([Phase 4]) | Yes | `test_backend_aca_env_drops_require_admin_auth_keeps_environment` present (L176-200); 39 passed |
| `v2/infra/main.json` | Yes (deviation note) | Yes | Untracked/gitignored compiled artifact; literal residue non-blocking |

No files were modified outside the changes-log inventory for Phase 4. No claimed Phase 4 change is missing from source.

## Recommended Next Validations (not performed this session)

* [ ] Phase 1 — Minimal `get_user_id` (dependencies.py rewrite + contract tests).
* [ ] Phase 2 — Admin routes → `UserIdDep`.
* [ ] Phase 3 — Delete the dead role-gate cluster + `require_admin_auth` + shared-gate exemption removal.
* [ ] Phase 5 — Documentation (bugs.md correction, worklog, ADR 0031, frontend no-op verification).
* [ ] Phase 6 — Validation (full backend + infra suite, `az bicep build` exit 0, live `/api/admin/status` 200 gated on user go-ahead).
* [ ] Optional cross-phase — confirm `az bicep build v2/infra/main.bicep` exits 0 locally (changes log reports exit 0; not re-run this session).

## Clarifying Questions

None. Phase 4 scope is self-contained and fully verifiable from source; all five requested checks pass.
