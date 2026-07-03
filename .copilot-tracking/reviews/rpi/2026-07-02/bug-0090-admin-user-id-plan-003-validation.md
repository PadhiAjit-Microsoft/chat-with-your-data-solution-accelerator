<!-- markdownlint-disable-file -->
# RPI Validation — BUG-0090 Phase 3: Delete the dead role-gate cluster + `require_admin_auth`

**Plan**: .copilot-tracking/plans/2026-07-02/bug-0090-admin-user-id-plan.instructions.md
**Details**: .copilot-tracking/details/2026-07-02/bug-0090-admin-user-id-details.md (Phase 3 = Lines 148-215)
**Changes log**: .copilot-tracking/changes/2026-07-02/bug-0090-admin-user-id-changes.md
**Research**: .copilot-tracking/research/2026-07-02/bug-0090-admin-401-user-id-research.md
**Phase**: 3 (Steps 3.1 + 3.2)
**Validation date**: 2026-07-02
**Status**: **Passed** (2 Minor nits, neither blocking)

## Scope

Phase 3 deletes the now-dead Easy Auth admin role-gate cluster from
`dependencies.py`, deletes the `require_admin_auth` field from
`settings.py`, rewrites the `types.py` docstrings that referenced the
deleted symbols, scrubs stale `local-dev`/401/Easy-Auth narrative from
`settings.py` / `history.py` / `conversation.py`, retains the
`environment` field + `Environment` enum, deletes the orphaned tests,
and removes the `_decode_easy_auth_principal` Hard-Rule-#15 allow-list
exemption. Validation is read + grep + `get_errors` only; no files were
modified.

## Requirement-by-requirement verdict

### R1 — ZERO deleted-symbol matches in `v2/src/**` (including comments/docstrings) — PASS

Grep over `v2/src/**` for
`requires_role|REQUIRE_ADMIN_USER|AdminUserIdDep|_decode_easy_auth_principal|_extract_roles|_LOCAL_DEV_USER|_PRINCIPAL_HEADER|_ROLE_TYP_SHORT|_ROLE_TYP_FULL|require_admin_auth`
returned **zero matches** (empty result). Spot-checked docstring sites
called out by the task:

* [v2/src/backend/core/settings.py](../../../../../v2/src/backend/core/settings.py#L519-L528) — the KEPT `environment` field comment is clean: it now describes the field as a "status-report field surfaced by `GET /api/admin/status`" with no `require_admin_auth` cross-reference and no auth-wall narrative.
* [v2/src/backend/core/settings.py](../../../../../v2/src/backend/core/settings.py#L41-L52) — the `Environment` enum docstring is clean (LOCAL / PRODUCTION mode discriminator only).
* [v2/src/backend/core/types.py](../../../../../v2/src/backend/core/types.py#L318-L322) — `RuntimeConfig.updated_by` docstring now reads "carries the admin caller's user id (from `get_user_id` / `UserIdDep` in `backend.dependencies`)". No `REQUIRE_ADMIN_USER` / `requires_role`.
* [v2/src/backend/core/types.py](../../../../../v2/src/backend/core/types.py#L347-L349) — `AdminAuditEntry.actor` docstring now references `get_user_id` / `UserIdDep`. Clean. (This is the changes-log "types.py:352" docstring; actual line ~348.)

The only surviving `require_admin_auth`-family token across the tree is the **string literal** `AZURE_REQUIRE_ADMIN_AUTH` inside [v2/tests/infra/test_main_bicep.py](../../../../../v2/tests/infra/test_main_bicep.py#L176-L192) (a Bicep env-var absence assertion), which the task explicitly instructs must NOT be flagged. Confirmed it is the Bicep binding name, not the deleted Python symbol. Not flagged.

### R2 — `environment` field + `Environment` enum RETAINED and referenced — PASS

* [v2/src/backend/core/settings.py](../../../../../v2/src/backend/core/settings.py#L528) — `environment: Environment = Environment.LOCAL` retained.
* [v2/src/backend/core/settings.py](../../../../../v2/src/backend/core/settings.py#L41) — `Environment` enum retained; exported in `__all__` at line 569.
* [v2/src/backend/routers/admin.py](../../../../../v2/src/backend/routers/admin.py#L128) — `environment=settings.environment` feeds `AdminStatus.environment`. Reference confirmed present.
* Retention is positively asserted by the surviving Environment-enum test cluster in [v2/tests/backend/core/test_settings.py](../../../../../v2/tests/backend/core/test_settings.py#L885-L958) (strenum-subclass, member-values, exactly-two-members, default-is-local, coerces-string, rejects-unknown).

### R3 — dead imports removed from `dependencies.py`; `get_errors` clean — PASS

[v2/src/backend/dependencies.py](../../../../../v2/src/backend/dependencies.py#L19-L35) imports are now: `logging`, `uuid`, `Annotated`, `AsyncTokenCredential`, `Depends, Request`, the provider base classes, `AppSettings, get_settings`, `ContentSafetyGuard`, `PostPromptValidator`, `RuntimeConfig`, `build_post_prompt_validator`. **None** of the task's dead-import list (`base64`, `binascii`, `json`, `Callable`, `Any`, `cast`, `HTTPException`, `status`, `Environment`) is present — the `fastapi` import is `Depends, Request` only, and `backend.core.settings` imports `AppSettings, get_settings` only. `get_errors` on `dependencies.py` returned **No errors found** → no now-undefined references. `__all__` was trimmed to the surviving symbol set (`UserIdDep` / `get_user_id` retained).

### R4 — no stale `local-dev` / 401 / Easy-Auth narrative in settings/history/conversation — PASS

* [v2/src/backend/routers/history.py](../../../../../v2/src/backend/routers/history.py#L8-L15) — module docstring positively rewritten to the default-GUID contract: a missing/blank/non-GUID header "folds into the anonymous default id `00000000-0000-0000-0000-000000000000` … which scopes a shared tenant partition rather than raising -- the id is a partition key, never a trust boundary." No `local-dev`, 401, or Easy Auth.
* [v2/src/backend/routers/conversation.py](../../../../../v2/src/backend/routers/conversation.py#L152-L153) — persistence comment positively rewritten: keyed by `user_id` "(the caller's principal id, or the anonymous default id when the header …)". No stale narrative.
* Targeted grep of `history.py` and `conversation.py` for `local-dev|401|Easy Auth|require_admin|principal claims|role gate` → **empty** for both.
* `get_errors` clean on `settings.py`, `history.py`, `conversation.py`.

### R5 — orphaned tests deleted + Hard-Rule-#15 allow-list entry removed — PASS

* Grep `v2/tests/**` for `_decode_easy_auth_principal|_extract_roles|test_requires_role|test_require_admin_auth` → **empty**. The `test_requires_role_*` suite, claims-helper tests, and the two `require_admin_auth` settings tests are gone.
* [v2/tests/backend/test_dependencies.py](../../../../../v2/tests/backend/test_dependencies.py#L10-L17) — the `backend.dependencies` import block imports only live symbols (`get_agents_provider`, `get_content_safety_guard`, `get_database_client`, `get_runtime_overrides`, `get_search_provider`, `get_user_id`); no `requires_role` / `REQUIRE_ADMIN_USER`.
* [v2/tests/backend/test_admin.py](../../../../../v2/tests/backend/test_admin.py#L193) — the fixture overrides `get_user_id` (`app.dependency_overrides[get_user_id] = lambda: _FIXED_USER_ID`); module docstring refreshed to the header-GUID contract ("there is no role gate"); no `REQUIRE_ADMIN_USER` / `require_admin_auth`.
* [v2/tests/shared/test_no_anonymous_dict_returns.py](../../../../../v2/tests/shared/test_no_anonymous_dict_returns.py#L67-L83) — the enforced `_ALLOWED` frozenset now has exactly 4 entries (`_request_extras`, `CosmosDBClient._read_item`, `FoundryIQ._to_openai_messages`, `_decode_event_payload`); the `backend.dependencies._decode_easy_auth_principal` tuple is removed. The corresponding docstring bullet was also removed. Grep of the file for `_decode_easy_auth_principal` / `easy_auth` / `dependencies` → empty.

## Findings (severity-ordered)

### Critical

None.

### Major

None.

### Minor

* **M-1 — Stale line-number citations for the `environment` reference (tracking-doc accuracy only).** The plan, details, and changes log all cite `admin.py:138` for `environment=settings.environment`; the actual reference lives at [v2/src/backend/routers/admin.py](../../../../../v2/src/backend/routers/admin.py#L128). The reference itself is correct and present — only the cited line number drifted (the docstring rewrite in Phase 2 shortened the module header). No code defect; the details file itself flags deletion as "grep-driven, not line-number-driven (validator DD-06)", so this is consistent with the plan's own posture. No action required for Phase 3 correctness.
* **M-2 — Cosmetic docstring-formatting artifact in the shared gate.** In [v2/tests/shared/test_no_anonymous_dict_returns.py](../../../../../v2/tests/shared/test_no_anonymous_dict_returns.py#L37-L41) the allow-list bullet list lost a line break where the deleted `_decode_easy_auth_principal` bullet used to sit: the `CosmosDBClient._read_item` bullet ends `…shape the SDK actually delivers.* ``backend.core.providers.llm.foundry_iq.FoundryIQ._to_openai_messages`` --`, joining the sentence terminator directly to the next bullet marker. This is inside a **test-file docstring** (not `v2/src/**`, so Hard Rule #16 does not apply) and does **not** affect the enforced `_ALLOWED` frozenset, which is correct. Purely cosmetic; a one-line-break insert would resolve it.

## Coverage assessment

Phase 3 is **fully implemented**. Every Step 3.1 and Step 3.2 success criterion is satisfied:

* Role-gate cluster + role-typ constants + `_LOCAL_DEV_USER` + `_PRINCIPAL_HEADER` deleted (grep-clean in `v2/src/**`, incl. docstrings). ✅
* `require_admin_auth` field + comment deleted; kept `environment` field comment scrubbed. ✅
* `types.py` `updated_by` + `AdminAuditEntry.actor` docstrings rewritten to `get_user_id`/`UserIdDep`. ✅
* Stale `local-dev`/401/Easy-Auth narrative scrubbed from `settings.py`, `history.py`, `conversation.py`. ✅
* `environment` field + `Environment` enum retained; `admin.py` still compiles + references the field. ✅
* No dead imports in `dependencies.py`; `get_errors` clean on all six source files. ✅
* Orphaned `test_requires_role_*` + two `require_admin_auth` tests deleted; `_decode_easy_auth_principal` exemption removed from both the docstring and the enforced frozenset. ✅

The changes log records `python -m pytest v2/tests/backend v2/tests/shared -q` green (2176 passed / 1 skipped; shared 1039 passed) at Phase 6; this validation did not re-execute the suite (read/grep/`get_errors` only per RPI protocol) but confirmed the static preconditions for those tests to pass.

## Recommended next validations (not completed this session)

* [ ] Validate Phase 4 (Bicep `AZURE_REQUIRE_ADMIN_AUTH` removal + `AZURE_ENVIRONMENT` retention) against `v2/infra/main.bicep` + `test_main_bicep.py`.
* [ ] Validate Phase 1 (`get_user_id` GUID contract) + Phase 2 (`AdminUserIdDep` → `UserIdDep` swap) test rewrites, since Phase 3 depends on them.
* [ ] Re-run `python -m pytest v2/tests/backend v2/tests/shared -q` to confirm the runtime-green claim in the changes log.
* [ ] Optionally correct M-2 (insert the missing docstring line break) and refresh the M-1 line citations to `admin.py:128` in the tracking docs.

## Clarifying questions

None. Phase 3 scope is fully covered by the available artifacts; both findings are Minor and non-blocking.
