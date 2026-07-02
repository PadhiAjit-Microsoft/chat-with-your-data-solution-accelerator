<!-- markdownlint-disable-file -->
# Research: WI-07 TS6133 Frontend Build Blocker

Status: Complete
Date: 2026-07-01

## Research scope

READ-ONLY investigation of the `TS6133` (declared-but-unused) build blocker in the
CWYD v2 frontend at `v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx`.
Questions answered below with exact file paths, line numbers, quoted code, and evidence.

No code was modified.

---

## Q1 — Exact TS6133 site

### The unused symbol

The unused symbol is the local helper `formatActor`, declared at
`v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx:628`:

```tsx
621  function formatTimestamp(value: string): string {
622    if (value === "") {
623      return "—";
624    }
625    return value;
626  }
627
628  function formatActor(value: string): string {
629    if (value === "") {
630      return "—";
631    }
632    return value;
633  }
```

`formatActor` is a trivial null/empty-string display formatter — identical shape to
its sibling `formatTimestamp` (line 621). It maps an empty actor string to an em-dash
`—`, otherwise returns the raw actor value (an Entra object ID string). It exists to
render the audit-footer "Updated by" value.

### Why it is unused — the commented-out "Updated by" audit render

The **only** call site of `formatActor` is inside a JSX comment. The audit footer
renders "Last updated" (active) but the "Updated by" `<p>` is commented out at
`Configuration.tsx:1040-1045`:

```tsx
1029                  {state.lastRuntime !== null ? (
1030                    <div
1031                      className={styles.auditFooter}
1032                      data-testid="config-audit-footer"
1033                    >
1034                      <p className={styles.auditLine}>
1035                        Last updated:{" "}
1036                        <span className={styles.auditValue}>
1037                          {formatTimestamp(state.lastRuntime.updated_at)}
1038                        </span>
1039                      </p>
1040                      {/* <p className={styles.auditLine}>
1041                        Updated by:{" "}
1042                        <span className={styles.auditValue}>
1043                          {formatActor(state.lastRuntime.updated_by)}
1044                        </span>
1045                      </p> */}
1046                    </div>
1047                  ) : null}
```

- Line 1037 (`formatTimestamp`) is **live** → `formatTimestamp` is used, no error.
- Lines 1040-1045 (`formatActor`) are **commented out** → `formatActor` has zero
  live references → `noUnusedLocals` fires `TS6133`.

### Why the compiler flags it — tsconfig

`v2/src/frontend/tsconfig.json:9` sets:

```json
"noUnusedLocals": true,
"noUnusedParameters": true,
```

The build script `v2/src/frontend/package.json:7` is `"build": "tsc -b && vite build"`.
`tsc -b` type-checks with `noUnusedLocals`, so the commented call site makes the
whole frontend build fail with:

```
Configuration.tsx(628): error TS6133: 'formatActor' is declared but never read.
```

Corroborating evidence in the tree (independent observers, all consistent):

- `v2/docs/worklog/2026-06-29.md:58` — "the frontend project-wide `tsc -b` is red on an
  **unused `formatActor`** in the open #35d admin-merge `Configuration.tsx`."
- `.copilot-tracking/plans/logs/2026-06-29/deletedata-page-changes-log.md:105` —
  "The symbol is dead because the 'Updated by' audit block (lines ~1040-1045) is
  commented out."
- `.copilot-tracking/plans/logs/2026-07-01/v2-containerize-services-and-model-cleanup-log.md:168`
  — WI-07: "Fix the pre-existing `TS6133` in `...Configuration.tsx:628` (`formatActor`
  unused; commented-out 'Updated by' audit line)."

---

## Q2 — Minimal-fix options

### Option (a) — UNCOMMENT the "Updated by" audit render *(the note's preferred fix)*

**Single edit site:** `Configuration.tsx:1040-1045`.

Remove the JSX comment wrapper so the block renders. Concretely:
- delete the `{/* ` opening at the start of line 1040, and
- delete the ` */}` closing at the end of line 1045.

Resulting live block:

```tsx
                      <p className={styles.auditLine}>
                        Updated by:{" "}
                        <span className={styles.auditValue}>
                          {formatActor(state.lastRuntime.updated_by)}
                        </span>
                      </p>
```

After this, `formatActor` (line 628) has a live call site → `TS6133` clears.

**Data-dependency check (does the render need a field that may not exist?): SAFE.**
The block reads `state.lastRuntime.updated_by`.

- `state.lastRuntime` is typed `RuntimeConfig | null` at
  `Configuration.tsx:306` (and the enclosing render is already guarded by
  `state.lastRuntime !== null` at line 1029).
- `RuntimeConfig.updated_by` exists and is a **non-null** `string`, declared at
  `v2/src/frontend/src/models/admin.tsx:278`:

  ```tsx
  278    updated_by: string;
  ```

  (Context: `RuntimeConfig` at `admin.tsx:270-281` carries server-set audit fields
  `updated_at: string` and `updated_by: string`, both non-null — "server-set audit
  fields the backend stamps on every successful PATCH.")

So uncommenting requires **no new field, no model change, no new import, no new CSS
class**. `styles.auditLine` and `styles.auditValue` are already used by the live
"Last updated" line (1034, 1036), so the class names resolve. The fix is one edit,
6 lines, zero new dependencies.

### Option (b) — REMOVE the unused `formatActor` helper

**Edit site:** delete `formatActor` at `Configuration.tsx:628-633` (6 lines).

**Dead-code sweep to make the removal clean:**
- The commented "Updated by" block at `Configuration.tsx:1040-1045` should ALSO be
  deleted (otherwise it references a now-deleted symbol inside a comment — harmless to
  the compiler but leaves misleading dead markup). This is what
  `.copilot-tracking/details/2026-06-29/deletedata-page-changes-details.md:19` calls
  "the dead `formatActor` / commented 'Updated by' block."
- No dead imports or state result from removal: `formatActor` pulls in nothing —
  it takes a `string`, returns a `string`, imports nothing. `state.lastRuntime`
  stays live (used by `formatTimestamp` at line 1037), so the reducer state field
  (`Configuration.tsx:306`, `362`, `425`, `484`) and the `RuntimeConfig` import are
  all still needed. Nothing else becomes orphaned.

**BUT option (b) does NOT fix the coupled vitest failure** — see Q3. Removing the
helper leaves the audit footer without a "Updated by" line, so the test that asserts
the footer contains `updated_by` continues to fail. To make option (b) green you would
*also* have to edit the failing test (delete its `updated_by` assertion), which is a
larger, behavior-reducing change.

### Which the tracking note prefers

Uncomment (option a). Rationale captured across the tracking logs:
- The WI-07 / IN-02 / ID-01 notes describe the block as an incomplete render left
  behind by the #35d admin merge, not intentionally-removed behavior.
- `.copilot-tracking/plans/logs/2026-06-26/cwyd-v2-networking-strategy-log.md` and the
  0629 logs all frame it as "resolve the dead `formatActor` + commented 'Updated by'
  audit line" — i.e. restore the intended audit line, which simultaneously satisfies
  the existing test.
- Uncommenting is the strictly smaller, behavior-restoring change and makes the
  already-written vitest assertion pass with no test edit.

---

## Q3 — Coupled vitest failure

### Exact failing test

- **File:** `v2/tests/frontend/pages/admin/Configuration/Configuration.test.tsx`
- **Test name (line 505):**
  `"surfaces the audit footer with the runtime updated_at / updated_by metadata after save"`
- **Enclosing suite:** the top-level `describe("Configuration ...")` block (save flow),
  distinct from the later `describe("Configuration -- reset to default")` block.

### The coupling assertion

`Configuration.test.tsx:505-525`:

```tsx
505    it("surfaces the audit footer with the runtime updated_at / updated_by metadata after save", async () => {
506      getMock.mockResolvedValueOnce(CONFIG_FIXTURE);
507      patchMock.mockResolvedValueOnce(RUNTIME_FIXTURE);
508      getMock.mockResolvedValueOnce(PATCHED_CONFIG_FIXTURE);
509
510      render(<Configuration />);
511
512      await waitFor(() => {
513        expect(screen.getByTestId("config-form")).toBeInTheDocument();
514      });
515      fireEvent.change(screen.getByTestId("config-input-orchestrator_name"), {
516        target: { value: "agent_framework" },
517      });
518      fireEvent.click(screen.getByTestId("config-save-button"));
519
520      await waitFor(() => {
521        expect(screen.getByTestId("config-audit-footer")).toBeInTheDocument();
522      });
523      const footer = screen.getByTestId("config-audit-footer");
524      expect(footer).toHaveTextContent("2026-06-03T11:00:00Z");
525      expect(footer).toHaveTextContent("admin-user-id");
526    });
```

The coupling is line **525**: `expect(footer).toHaveTextContent("admin-user-id")`.

The fixture feeding the PATCH response is `RUNTIME_FIXTURE`
(`Configuration.test.tsx:64-79`):

```tsx
64   const RUNTIME_FIXTURE: RuntimeConfig = {
     ...
77     updated_at: "2026-06-03T11:00:00Z",
78     updated_by: "admin-user-id",
79   };
```

- Line 524 asserts the footer contains `"2026-06-03T11:00:00Z"` (the `updated_at`) —
  this **passes today** because the "Last updated" render at `Configuration.tsx:1037`
  is live.
- Line 525 asserts the footer contains `"admin-user-id"` (the `updated_by`) —
  this **fails today** because the only element that would render `updated_by` is the
  commented-out "Updated by" block at `Configuration.tsx:1040-1045`. The text
  `admin-user-id` never enters the DOM.

### Does uncommenting make it pass? YES.

Uncommenting `Configuration.tsx:1040-1045` renders
`{formatActor(state.lastRuntime.updated_by)}` → `formatActor("admin-user-id")` →
`"admin-user-id"` into a `<span className={styles.auditValue}>` inside the
`config-audit-footer` div. `expect(footer).toHaveTextContent("admin-user-id")` then
matches. Uncommenting fixes **both** the `TS6133` (Q1) and this vitest assertion in a
single edit — exactly as the tracking note states.

(Note: a second, unrelated fixture `DEFAULTS_RUNTIME` at
`Configuration.test.tsx:1071-1088` also sets `updated_by: "admin-user-id"`, but its
suite `"Configuration -- reset to default"` does not assert on the "Updated by" footer
text, so it is not the failing test and is unaffected either way.)

---

## Q4 — #35d coupling risk

### What #35d is, and its current status

`#35d` is the "Streamlit → React admin merge" work item. The commented "Updated by"
block was left behind while that admin surface (including `Configuration.tsx`) was
merged.

**Key finding — #35d is reported CLOSED in the status docs, despite stale "open"
language in some tracking logs:**

- `v2/docs/mvp_status.md:161` — "The Streamlit→React admin merge (`#35d`) is cleared."
- `v2/docs/project_status.md:25` — "`#35d` Streamlit-to-React admin merge is ✅ cleared
  (2026-06-08, `U-P7-35D-AUDIT`)."
- `v2/docs/project_status.md:116` — "FE admin route merge | ✅ cleared (`#35d`, 2026-06-08)".
- `v2/docs/project_status.md:142` — "FE merge `#35d` cleared 2026-06-08".
- `v2/docs/adr/0009-single-owner-no-separate-team-framing.md:8` supersedes the old
  "separate frontend team" framing on the #35d rows — so the "owned by the admin-merge
  owner / FE team, leave it for them" language in the worklogs is itself obsolete
  process framing, not a live blocker.

Conversely, the worklogs still colloquially call it "the open #35d admin-merge
`Configuration.tsx`" (`v2/docs/worklog/2026-06-29.md:58`;
`.copilot-tracking/plans/logs/2026-06-29/auth-enforced-removal-transparent-identity-log.md:85`).
This is stale terminology relative to the status docs, which mark #35d cleared.

### Is the fix entangled with unmerged #35d changes? NO — it is self-contained.

Evidence the change touches nothing beyond the two adjacent regions of one file:

1. **All symbols the uncommented block needs already exist and are live/committed:**
   - `formatActor` — `Configuration.tsx:628` (committed).
   - `state.lastRuntime` — reducer state `RuntimeConfig | null`, `Configuration.tsx:306`;
     already dereferenced by the live "Last updated" line at 1037.
   - `RuntimeConfig.updated_by: string` — `admin.tsx:278` (committed, non-null).
   - `styles.auditLine` / `styles.auditValue` — already consumed by the live line
     (1034, 1036), so the CSS module exports them.
2. **No half-merged data path.** The PATCH → `RuntimeConfig` audit fields
   (`updated_at`/`updated_by`) are fully wired end-to-end: backend
   `v2/src/backend/core/types.py:319`, router `v2/src/backend/routers/admin.py:236`,
   model mirror `admin.tsx:270-281`, and the test fixture
   (`Configuration.test.tsx:64-79`). Nothing about `updated_by` is pending.
3. **The prior task that touched this area explicitly scoped it OUT, not because it
   was risky/entangled, but to keep that task's diff minimal:**
   `.copilot-tracking/details/2026-06-29/deletedata-page-changes-details.md:19` —
   "Do NOT touch the dead `formatActor` / commented 'Updated by' block in
   Configuration.tsx (out of scope)." That is a scope guard for a *different* task
   (DeleteData page), not evidence of a merge conflict.

**Residual #35d note (not a blocker for this fix):** #35d follow-on work exists around
the prompt-editor route (`PromptEditor.tsx`) and is referenced by `BUG` entries
(`v2/docs/bugs.md:261`), but that is a *different file and route* and does not touch
`Configuration.tsx`'s audit footer. The WI-07 fix does not depend on, and is not
blocked by, any PromptEditor work.

**Conclusion:** the fix is self-contained — one file, two adjacent regions (uncomment
1040-1045; helper at 628 becomes used). No unmerged #35d change gates it. The
"belongs to #35d owner" framing in the logs is stale ownership language, superseded by
ADR 0009 and by the status docs marking #35d cleared.

---

## Q5 — Verification commands

All commands run from `v2/src/frontend/` unless noted. Scripts confirmed from
`v2/src/frontend/package.json`, `v2/tests/frontend/package.json`, and `v2/package.json`.

### Reproduce the TS6133 (build blocker)

From `v2/src/frontend/`:

```powershell
# Full production build (the exact path the Docker prod image runs):
npm run build          # → runs "tsc -b && vite build"; fails at tsc -b

# Type-check only (isolate the TS6133 without the vite bundling step):
npx tsc -b             # matches the build script's type-check invocation
# or, plain check (tsconfig already sets noEmit:true):
npx tsc --noEmit
```

Expected pre-fix failure:

```
src/pages/admin/Configuration/Configuration.tsx(628,10): error TS6133: 'formatActor' is declared but never read.
```

The frontend prod Docker image reproduces the same failure because
`v2/docker/Dockerfile.frontend --target prod` runs `npm run build` internally.

Equivalent from the `v2/` workspace root (npm workspaces):

```powershell
# from v2/
npm run build          # → "npm run build --workspace cwyd-frontend"
```

### Run the coupled vitest test

The frontend test tree is a **separate workspace** (`cwyd-frontend-tests` under
`v2/tests/frontend/`), run with `vitest run`.

From `v2/` (workspace root):

```powershell
npm test               # → "npm run test --workspace cwyd-frontend-tests" → "vitest run"
```

From `v2/tests/frontend/` (the test package directly):

```powershell
npm test               # → "vitest run"

# Narrow to just the Configuration suite + the failing case:
npx vitest run pages/admin/Configuration/Configuration.test.tsx -t "surfaces the audit footer"
```

Expected pre-fix failure (the `updated_by` assertion, `Configuration.test.tsx:525`):

```
FAIL  pages/admin/Configuration/Configuration.test.tsx > Configuration > surfaces the audit footer with the runtime updated_at / updated_by metadata after save
  Unable to find text content "admin-user-id" in the config-audit-footer element
```

After uncommenting `Configuration.tsx:1040-1045`, both `npm run build` (TS6133 gone)
and the vitest case (footer now contains `admin-user-id`) go green.

---

## RECOMMENDATION

**Uncomment the "Updated by" audit render** at
`v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx:1040-1045` (delete the
`{/* ` at the head of line 1040 and the ` */}` at the tail of line 1045).

Rationale:
1. **Smallest change that clears both blockers.** One edit fixes the `TS6133` (Q1) and
   the coupled vitest assertion (Q3) simultaneously, exactly as the tracking note
   states. Removing `formatActor` (option b) fixes only the compiler error and would
   *additionally* require editing/weakening the existing test — a larger,
   behavior-reducing change.
2. **Data-safe, dependency-free.** The render reads `state.lastRuntime.updated_by`,
   which is a committed, non-null `string` on `RuntimeConfig` (`admin.tsx:278`) and is
   already guarded by the `state.lastRuntime !== null` check at line 1029. No model
   change, no new import, no new CSS class — the CSS classes are already used by the
   live "Last updated" line.
3. **Restores intended behavior.** The audit footer was designed to show both "Last
   updated" and "Updated by"; the "Updated by" half was merely left commented during
   the #35d merge. Uncommenting completes the intended UI.
4. **Self-contained.** No entanglement with unmerged #35d work (Q4); #35d is marked
   cleared in the status docs (2026-06-08), and the only symbols involved live entirely
   within `Configuration.tsx` + the already-committed `RuntimeConfig` model.

### Residual risk (low)

- **UI/visual regression:** uncommenting adds a visible "Updated by: &lt;Entra object
  ID&gt;" line to the admin Configuration footer. This surfaces a raw Entra object ID
  string to admins (not a secret; already stored server-side; already shipped in the
  `RuntimeConfig` wire model). Acceptable for an admin-only page, but worth a glance in
  the running UI if brand/UX polish is a concern.
- **Snapshot/CSS drift:** none expected — no snapshot tests reference the footer;
  `styles.auditLine`/`styles.auditValue` already exist. The compiled bundle
  `v2/src/frontend/dist/assets/index-Ca-035km.js:94` already contains a built "Updated
  by:" render from an earlier build, confirming the class names and structure are valid.
- **Stale tracking language:** logs still call this "open #35d" work "owned by the
  admin-merge owner." Per ADR 0009 and the status docs, #35d is cleared and that
  ownership framing is obsolete — so acting on WI-07 does not step on an active owner.
- **Two-region touch:** the edit is one file but if a future task instead chose removal
  (option b), remember to also delete the commented block AND the test assertion —
  option (a) avoids that coordination entirely.

---

## Evidence index (files read)

- `v2/src/frontend/src/pages/admin/Configuration/Configuration.tsx` — lines 560-700,
  990-1080, 1027-1049 (helpers 621/628; commented block 1040-1045; state type 306).
- `v2/src/frontend/src/models/admin.tsx` — lines 1-320 (`RuntimeConfig.updated_by`
  at 278; `EffectiveAdminConfig` audit fields).
- `v2/tests/frontend/pages/admin/Configuration/Configuration.test.tsx` — lines 55-90
  (`RUNTIME_FIXTURE`), 490-545 (failing test 505 + assertions 524-525), 1070-1130
  (`DEFAULTS_RUNTIME`, unrelated suite).
- `v2/src/frontend/tsconfig.json` — `noUnusedLocals: true` (line 9).
- `v2/src/frontend/package.json` — `"build": "tsc -b && vite build"`.
- `v2/tests/frontend/package.json` — `"test": "vitest run"`.
- `v2/tests/frontend/vitest.config.ts` — jsdom, globals, include glob.
- `v2/package.json` — workspace scripts (`build`, `test`, `lint`).
- Tracking / docs: `v2/docs/worklog/2026-06-29.md:58`; `v2/docs/mvp_status.md:161`;
  `v2/docs/project_status.md:25,116,142`; `v2/docs/adr/0009-...md:8`;
  `v2/docs/bugs.md:261`;
  `.copilot-tracking/plans/logs/2026-06-29/deletedata-page-changes-log.md:90,104-105`;
  `.copilot-tracking/details/2026-06-29/deletedata-page-changes-details.md:19`;
  `.copilot-tracking/plans/logs/2026-07-01/v2-containerize-services-and-model-cleanup-log.md:168`.

## Clarifying questions

None — all five research questions are answered with in-tree evidence. (One minor
observation for the implementer: the tracking logs disagree with the status docs on
whether #35d is "open" vs "cleared"; the status docs win, and either way the WI-07 fix
is self-contained.)

## Recommended next research (not done here)

- [ ] None required to execute WI-07. Optional: confirm the running admin
      Configuration page renders the restored "Updated by" line acceptably (visual
      check) after the uncomment lands.
