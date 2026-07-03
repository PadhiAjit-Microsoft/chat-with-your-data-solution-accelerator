# BUG-0055 — infra test + ADR impact research

Status: Complete
Date: 2026-07-02
Scope: READ-ONLY. Impact of changing `v2/infra/main.bicep` App Insights module
`disableLocalAuth: true` → `false` (~line 326) and possibly REMOVING the
`Monitoring Metrics Publisher` role assignment (~lines 330–336).

Placeholders per Hard Rule #18 / user memory: `<SUFFIX>` used for any Azure ids.

---

## Research questions

1. Any test/script asserting on `disableLocalAuth` (grep `disableLocalAuth`,
   `disable_local_auth`, `DisableLocalAuth`) — does it assert value `true` for
   App Insights specifically?
2. Any test/guard referencing `Monitoring Metrics Publisher` /
   `3913510d-42f4-4e42-8a64-81b1edca285c` / `roleAssignments` on App Insights?
3. Is there an infra test harness validating `main.bicep`? How does it validate
   (string parse vs compiled JSON vs `az bicep build` subprocess)? Pattern to follow.
4. ADR-0018 structure — amendment heading pattern; current `disableLocalAuth` +
   `Monitoring Metrics Publisher` statements.
5. BUG-0055 registry row exact text + line.
6. Is `disableLocalAuth` a Bicep `param` or inline literal? Is `enableMonitoring`?

---

## (a) Tests/guards asserting `disableLocalAuth`

Answer: **NO test asserts `disableLocalAuth: true` (or any value) on the App
Insights component.** All hits are in comments / assertion-message prose only —
none is an executable assertion on the `disableLocalAuth` literal.

Whole-tree grep results (plain-text, `v2/`):

- Only one TEST file matches: `v2/tests/infra/test_main_bicep.py`
  - Line 321 — module-level comment: `# `disableLocalAuth: true`, so ingestion authenticates via Entra. Without`
  - Line 353 — inside the `Monitoring Metrics Publisher` assertion **message string** (not the assert expression): `"with disableLocalAuth=true, so without this role telemetry "`
  - Line 636 — comment on the **aiSearch** slice: `# aiSearch service sets disableLocalAuth=true (RBAC-only data plane), so`
  - Line 658 — assertion-message prose on the **aiSearch** slice: `with `disableLocalAuth: true` the data plane is RBAC-only,`
  - None of these four is an `assert "disableLocalAuth" ...`. The token never appears in an assert expression anywhere in `v2/tests/`.

- `v2/infra/main.bicep` — the literal itself (not a test):
  - Line 326 — App Insights: `disableLocalAuth: true` (the target of the planned change)
  - Lines 521, 723, 798, 861 — sibling data-plane modules (aiServices, aiSearch, storage, etc.) each set `disableLocalAuth: true` independently. The planned change touches ONLY line 326 (App Insights); these are out of scope.

- `v2/infra/main.json` — compiled ARM output (build artifact, NOT a test). Multiple
  `disableLocalAuth` hits are inside AVM sub-templates. No pytest reads `main.json`
  (grep `main.json` in `v2/tests/**` → 0 hits), so there is no compiled-JSON snapshot test.

Impact: flipping `disableLocalAuth: true → false` on App Insights (line 326)
trips **no direct value assertion**. But it invalidates the *rationale prose* in
the message string at line 353 (which explains why the role is needed "with
disableLocalAuth=true") — that prose should be updated for accuracy even though
it will not fail the run by itself. See (b) for the assertion that WILL fail.

---

## (b) Tests/guards referencing `Monitoring Metrics Publisher` role (App Insights)

Answer: **YES — one test asserts the App Insights role assignment and it WILL
FAIL if the role is removed.** Path: `v2/tests/infra/test_main_bicep.py`.

Test function: `test_application_insights_grants_metrics_publisher_to_uami`
(defined ~line 347), operating on the `application_insights_slice` fixture.

Grep hits (`Monitoring Metrics Publisher`, `v2/tests/`):
- Line 318 — comment header `# ADR-0018: Monitoring Metrics Publisher RBAC for UAMI on AppI.`
- Line 322 — comment prose
- Line 333 — constant: `_MONITORING_METRICS_PUBLISHER_ROLE_NAME = "Monitoring Metrics Publisher"`
- Line 349 — docstring: `"""The AppI module must grant `Monitoring Metrics Publisher` to the UAMI (ADR-0018)."""`

The role-definition GUID `3913510d-42f4-4e42-8a64-81b1edca285c` appears in tests: **0 hits**
(it only appears in `v2/docs/adr/0018-*.md` lines 21, 30, 112). The test matches
on the role NAME, not the GUID, because the AVM module resolves the built-in name.

The three assertions in that test (all on `application_insights_slice`):

```python
assert "roleAssignments:" in application_insights_slice, (...)              # ~line 350
assert _MONITORING_METRICS_PUBLISHER_ROLE_NAME in application_insights_slice, (...)  # ~line 356
assert "userAssignedIdentity.outputs.principalId" in application_insights_slice, (...)  # ~line 361
```

The `application_insights_slice` fixture (~line 340):

```python
@pytest.fixture(scope="module")
def application_insights_slice(bicep_text: str) -> str:
    """Bicep source between `module applicationInsights` and the next section."""
    return _slice_module(
        bicep_text,
        "module applicationInsights ",
        "// Virtual network ",
    )
```

Impact: if the plan REMOVES the `roleAssignments` block from the App Insights
module, ALL THREE assertions fail (there is no longer a `roleAssignments:` key,
no `Monitoring Metrics Publisher` string, and no `principalId` reference inside
the slice). **This test must be removed or rewritten in the same turn as the
Bicep change.** The `_MONITORING_METRICS_PUBLISHER_ROLE_NAME` constant (line 333)
and the ADR-0018 comment header (lines 318–331) become dead and should be removed too.

Other `roleAssignments` assertions in the same file (lines 614–669) target
`storageAccount` and `aiSearch` slices — NOT App Insights — so they are unaffected.

---

## (c) Infra test harness for `main.bicep`

Answer: **YES.** File: `v2/tests/infra/test_main_bicep.py`.

Directory `v2/tests/infra/` contents:
- `test_main_bicep.py` (the main.bicep drift-guard harness)
- `test_ai_project_search_connection.py` (sibling module harness, same style)
- `__init__.py`

How it validates: **pure Python string-parse — NO subprocess, NO compiled JSON.**
- Reads the raw Bicep text once:
  ```python
  _BICEP = Path(__file__).resolve().parents[2] / "infra" / "main.bicep"

  @pytest.fixture(scope="module")
  def bicep_text() -> str:
      return _BICEP.read_text(encoding="utf-8")
  ```
- Slices a single `module ... { ... }` block by start/end markers:
  ```python
  def _slice_module(text: str, start_marker: str, end_marker: str) -> str:
      start = text.find(start_marker)
      assert start != -1, ...
      end = text.find(end_marker, start + len(start_marker))
      assert end != -1, ...
      return text[start:end]
  ```
- Each test asserts substring presence (`"<x>" in slice`) or absence (`"<x>" not in bicep_text`).
- The module docstring explicitly states the full contract "is validated by
  `az bicep build` (run as the last step of CU-009a)" — i.e. `az bicep build` is
  a MANUAL step, NOT automated in pytest. Grep confirms: `az bicep build` appears
  only in docstrings (lines 3 + the sibling file line 11); the only `subprocess`
  usages in `v2/tests/` are in unrelated files (`test_post_provision.py`,
  `test_no_env_specific_content.py`), none for bicep compilation.

Pattern a new/updated assertion follows (for this change): slice the App Insights
module via the `application_insights_slice` fixture, then assert a substring is
PRESENT (positive guard) or ABSENT (negative guard). Example negative-guard shape
(if the plan wants to pin the role's REMOVAL rather than delete the test):

```python
def test_application_insights_omits_metrics_publisher_role(application_insights_slice: str) -> None:
    assert "Monitoring Metrics Publisher" not in application_insights_slice, (...)
    assert "disableLocalAuth: false" in application_insights_slice, (...)  # optional value pin
```

Note: the harness slices by literal markers, so if the change alters the module's
neighbouring text the markers (`"module applicationInsights "`, `"// Virtual network "`)
must still bracket the block — they currently do and the planned edit does not touch them.

---

## (d) ADR-0018 structure + amendment heading pattern

File: `v2/docs/adr/0018-monitoring-default-on-and-appi-rbac.md`.

Front matter / section order:
- H1: `# ADR 0018 — Monitoring default-on for deployed envs + `Monitoring Metrics Publisher` RBAC for UAMI on AppI`
- Metadata bullets: `**Status**: Accepted`, `**Date**: 2026-06-05`, `**Phase**: 7`, `**Pillar**: Stable Core`, `**Deciders**`
- `## Context`
- `## Decision` (numbered 1–3) + `### Wire shape (binding)` + `### Out of scope — what this ADR does NOT decide`
- `## Consequences` (`### Positive`, `### Negative`, `### Neutral`)
- `## Alternatives considered` (numbered 1–5)
- `## Amendment 1 (2026-06-23) — per-workload App Insights env-var names (BUG-0055)`
- `## References`

Amendment heading pattern (EXACT, to match for a new amendment):

```
## Amendment 1 (2026-06-23) — per-workload App Insights env-var names (BUG-0055)
```

So a new amendment should be:

```
## Amendment 2 (<YYYY-MM-DD>) — <short title> (BUG-0055)
```

placed AFTER "Amendment 1" and BEFORE "## References". Body is free prose
paragraphs (no fixed sub-structure); Amendment 1 uses a lead paragraph +
"Resolution — ..." bullet list.

Current ADR-0018 statements the planned change CONTRADICTS (a new amendment must
address these):

- Context (¶ starting "There is also a **second, latent gap**"): frames
  `disableLocalAuth: true` (cited as "line 320 of main.bicep") as "the
  WAF-aligned setting — the data-plane refuses instrumentation-key auth and
  requires Microsoft Entra ID tokens" and the missing role as the silent-401 gap.
- Decision point 2: "**The Bicep MUST assign `Monitoring Metrics Publisher`**
  (`3913510d-42f4-4e42-8a64-81b1edca285c`) to the UAMI on the `applicationInsights`
  resource scope ... the UAMI-based OpenTelemetry exporter (the only ingestion
  path because `disableLocalAuth: true`) succeeds."
- Consequences → Positive: "**UAMI + Entra-only ingestion stays.**
  `disableLocalAuth: true` is preserved ... This is the WAF-aligned path."
- Alternatives considered #3: "**Default-on monitoring but skip the
  `Monitoring Metrics Publisher` role assignment.** Rejected: the silent-401
  ingestion drop is the exact failure mode this ADR exists to close."

So flipping to `disableLocalAuth: false` and removing the role directly reverses
Decision #2 + Positive consequence + rejected-Alternative #3 → a new
`## Amendment 2` is required to record the reversal and its rationale (BUG-0055
"App Insights receives zero telemetry" — the Entra-only path never ingested).

---

## (e) BUG-0055 registry row — exact text + line

Registry table header (line 57):

```
| ID | Found | Fixed | Area | Severity | Status | Summary |
```

BUG-0055 registry row — **line 114**:

```
| BUG-0055 | 2026-06-16 | — | infra | medium | open | Application Insights (`appi-<SUFFIX>`) has received **zero** telemetry ever from both the function host and the backend Container App. A union query over `requests`/`traces`/`exceptions`/`dependencies`/`customMetrics` summarizing `count`/`min(timestamp)`/`max(timestamp)` returned `[0, null, null]` — total 0, no earliest, no latest. Both runtimes carry `APPLICATIONINSIGHTS_CONNECTION_STRING`, yet nothing arrives, so the OpenTelemetry / App Insights export is unwired or misconfigured at both runtimes (wrong connection string, missing exporter init, or a transport/sampling failure). Independently a defect (no production observability) and it blocks diagnosis of BUG-0053 (no scale-controller / host logs to inspect). |
```

Cells: `ID=BUG-0055 | Found=2026-06-16 | Fixed=— | Area=infra | Severity=medium |
Status=open | Summary=...`. Status is currently `open`; the plan will later flip
`Status → fixed` and set the `Fixed` date, and append the fix to the Summary.

There is also a detail section for BUG-0055 (H3 heading) at line 998:
`### BUG-0055 — Application Insights receives zero telemetry from the function + backend`
(the fuller narrative lives there; the plan should update both the row at line 114
and the detail section at line 998 when marking fixed).

Cross-references to BUG-0055 elsewhere in bugs.md: lines 112, 139, 147, 968, 970,
1330 (mentions from sibling bugs — no edit needed for the fix, informational only).

---

## (f) `disableLocalAuth` — param or inline literal? `enableMonitoring`?

- `disableLocalAuth` on App Insights (main.bicep line 326): **INLINE LITERAL** —
  hard-coded `disableLocalAuth: true` on the `applicationInsights` AVM module's
  `params:` object. It is NOT a `param` on `main.bicep`'s own parameter surface;
  there is no `param disableLocalAuth` declaration in `main.bicep`. (The
  `disableLocalAuth` param entries in `main.json` are inside the AVM sub-module's
  own compiled template, not main.bicep's interface.) → The planned change edits
  a literal value in place: `disableLocalAuth: true` → `disableLocalAuth: false`.

- The `roleAssignments` block (lines ~330–336) is also an INLINE array literal on
  the same module's `params:`:
  ```bicep
  roleAssignments: [
    {
      principalId: userAssignedIdentity.outputs.principalId
      principalType: 'ServicePrincipal'
      roleDefinitionIdOrName: 'Monitoring Metrics Publisher'
    }
  ]
  ```
  → REMOVING it is an inline deletion, not a param default change.

- `enableMonitoring`: **IS a Bicep `param`** — `main.bicep` line 195:
  ```bicep
  param enableMonitoring bool = true
  ```
  (default flipped to `true` by ADR-0018). The whole App Insights module is gated
  `= if (enableMonitoring)`. The planned change does NOT touch `enableMonitoring`.

---

## Answers to the caller's direct questions

- Does any test assert `disableLocalAuth: true` on App Insights? **NO.** The token
  appears only in comments + assertion-message prose in
  `v2/tests/infra/test_main_bicep.py` (lines 321, 353, 636, 658); never in an
  assert expression. No compiled-JSON snapshot test reads `main.json`.
- Does any test assert the `Monitoring Metrics Publisher` role? **YES.**
  `v2/tests/infra/test_main_bicep.py` →
  `test_application_insights_grants_metrics_publisher_to_uami` (~line 347) asserts
  3 substrings on the App Insights slice (`roleAssignments:`,
  `Monitoring Metrics Publisher`, `userAssignedIdentity.outputs.principalId`).
  Removing the role → this test fails and must be updated/removed in the same turn.
- Is there a main.bicep test harness? **YES** —
  `v2/tests/infra/test_main_bicep.py`. Validation = pure Python string-parse
  (`read_text` + `_slice_module` substring find + `assert "<x>" in slice`). No
  `az bicep build` subprocess, no compiled-JSON snapshot.
- ADR-0018 amendment heading pattern:
  `## Amendment 1 (2026-06-23) — per-workload App Insights env-var names (BUG-0055)`
  → next is `## Amendment 2 (<YYYY-MM-DD>) — <title> (BUG-0055)`, placed before
  `## References`.
- BUG-0055 registry row line number: **line 114** (header at line 57; detail
  section at line 998).

---

## Recommended next research (not done this session)

- [ ] Confirm whether the plan wants to DELETE `test_application_insights_grants_metrics_publisher_to_uami` outright or REPLACE it with a negative guard (`"Monitoring Metrics Publisher" not in slice`) — a product decision, not discoverable by research.
- [ ] Check `v2/docs/development_plan.md` §0.1 for any one-line pointer row referencing BUG-0055 that would also need updating on fix (Hard Rule #12 defect/debt split).
- [ ] Confirm the sibling data-plane `disableLocalAuth: true` literals (main.bicep lines 521/723/798/861) are intentionally OUT of scope for BUG-0055 (they cover aiServices/aiSearch/storage RBAC data planes, unrelated to App Insights ingestion).

## Clarifying questions

- None blocking. The one open product decision (delete vs invert the role-assertion
  test) is noted above for the planner.
