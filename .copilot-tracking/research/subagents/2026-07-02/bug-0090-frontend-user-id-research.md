<!-- markdownlint-disable-file -->

# BUG-0090 — CWYD v2 Frontend `user_id` + initials research

## Research topics / questions

Document how the CWYD v2 React/Vite frontend (`v2/src/frontend`) obtains and
sends `user_id` (and user initials) to the backend.

Intended behavior per the user:

- Frontend auth (identity provider) ENABLED → collect real `user_id` + initials.
- Frontend auth DISABLED → `user_id` defaults to a GUID, initial is `G`.
- The frontend ALWAYS passes `user_id` in request headers (default or real).

Verify what actually exists today. Specific questions:

1. API client — where it is, whether it sends a user-identity header.
2. `user_id` source + default GUID — where derived, default-GUID generation, initials, `/.auth/me` vs MSAL.
3. Auth enable/disable flag — any FE env var/config toggle.
4. Current user display / initials — the avatar/initials component, how `G` fallback is produced.
5. Models/types — TS shapes holding `userId`; is `user_id` in body or headers.

## Status

Complete.

## Executive summary (answers up front)

- **Does the FE send a user-id header today? YES.** Every user-facing REST/SSE
  client spreads `userIdHeaders()` which emits the header
  `x-ms-client-principal-id: <resolved id>`. It is sent as a **request HEADER**,
  never in the request body.
- **Default GUID: it is a FIXED all-zeros sentinel, NOT a generated GUID.**
  `DEFAULT_USER_ID = "00000000-0000-0000-0000-000000000000"`. The frontend does
  **not** call `crypto.randomUUID()` for the anonymous/default user — it forwards
  the same all-zeros id for every unauthenticated session. (Note: the FE *does*
  use `crypto.randomUUID()` elsewhere — for chat message ids and ingest batch
  ids — but never for `user_id`.)
- **`G` initial: works, via a "Guest" display-name fallback.** When no user is
  resolved, `resolveDisplayName(null)` returns `"Guest"`, and
  `userInitials("Guest")` returns `"G"`. `userInitials("")` also returns `"G"`.
- **Auth enable/disable flag: there is NO explicit frontend toggle.** Auth is
  *implicit* — driven entirely by whether the Easy Auth `/.auth/me` endpoint
  returns a principal. The only `VITE_*` variable in the FE is `VITE_BACKEND_URL`
  (backend origin), not an auth switch. There is no MSAL flow.
- **Real user id source:** Easy Auth `/.auth/me` on the SPA's own origin →
  the Entra object-identifier (`oid`) claim. No MSAL, no direct AAD call.

### Discrepancy vs the intended behavior (likely BUG-0090 crux)

| Intended | Actual today | Match? |
| --- | --- | --- |
| Auth ENABLED → real `user_id` (oid) + real initials | `/.auth/me` → `oid` claim; initials from `name` claim | ✅ matches |
| Auth DISABLED → `user_id` "defaults to a GUID" | `user_id` = fixed all-zeros sentinel `0000...0000` (shared by all anon users), NOT a per-session random GUID | ⚠️ partial — it is GUID-*shaped* but a shared sentinel, not a unique generated GUID |
| Auth DISABLED → initial `G` | `resolveDisplayName(null)` → `"Guest"` → `userInitials` → `"G"` | ✅ matches |
| Always send `user_id` in headers | `x-ms-client-principal-id` always spread onto every user-facing request | ✅ matches |

The one behavioral gap: if BUG-0090 expects a **unique** GUID per anonymous
session/user (via `crypto.randomUUID()`), today's code instead forwards a
**single shared all-zeros partition key** for every unauthenticated caller. All
anonymous users therefore share one chat-history partition on the backend.

---

## Q1 — API client(s) + user-identity header

### Files (all under `v2/src/frontend/src/api/`)

- `v2/src/frontend/src/api/auth.tsx` — identity resolution + the header builder (the single seam).
- `v2/src/frontend/src/api/streamChat.tsx` — `POST /api/conversation` SSE client.
- `v2/src/frontend/src/api/conversationHistory.tsx` — `GET /api/history/conversations/{id}`.
- `v2/src/frontend/src/api/admin.tsx` — admin REST calls (multiple endpoints).
- `v2/src/frontend/src/api/speech.tsx` — Speech token mint.
- `v2/src/frontend/src/api/runtimeConfig.tsx` — `/config` fetch; **does NOT** send the user header (no user needed to read runtime config).

There is **no** generated OpenAPI client for these calls; they are hand-written `fetch` wrappers. (`getBackendUrl()` from `runtimeConfig.tsx` supplies the origin.)

### The header builder — `v2/src/frontend/src/api/auth.tsx`

Line 33 defines the header name; lines 102-107 build it:

```tsx
// line 33
const PRINCIPAL_ID_HEADER = "x-ms-client-principal-id";
```

```tsx
// lines ~100-107
export function userIdHeaders(): Record<string, string> {
  return { [PRINCIPAL_ID_HEADER]: getUserId() };
}
```

Docstring (auth.tsx lines 28-33) is explicit that this is **not** a trust boundary:

> Identity header every API client forwards for per-user partitioning.
> A browser-set value is forgeable and is **not** a trust boundary — it
> scopes chat history only; admin RBAC stays anchored on the backend's
> own server-injected Easy Auth claims.

### Header consumers (each spreads `...userIdHeaders()`)

`v2/src/frontend/src/api/streamChat.tsx` lines 244-250:

```tsx
    response = await fetch(conversationUrl(), {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Accept: "text/event-stream",
        ...userIdHeaders(),
      },
```

`v2/src/frontend/src/api/conversationHistory.tsx` line 146:

```tsx
    headers: { Accept: "application/json", ...userIdHeaders() },
```

`v2/src/frontend/src/api/speech.tsx` line 56:

```tsx
    headers: { Accept: "application/json", ...userIdHeaders() },
```

`v2/src/frontend/src/api/admin.tsx` — imports it at line 26 and spreads it on
every call: lines 120, 147, 175, 198, 223, 252, 277 (e.g. line 120):

```tsx
    headers: { Accept: "application/json", ...userIdHeaders() },
```

**Conclusion:** the frontend **does** send a user-identity header today
(`x-ms-client-principal-id`) on every user-facing request. No `Authorization`
bearer header, no `X-User`, no body-embedded id.

---

## Q2 — `user_id` source + default GUID

### Real user id — Easy Auth `/.auth/me` (no MSAL)

`v2/src/frontend/src/api/auth.tsx` lines 20-25 define the claim URI, lines 60-83
resolve it:

```tsx
// lines 20-24
const OBJECT_ID_CLAIM =
  "http://schemas.microsoft.com/identity/claims/objectidentifier";
```

```tsx
// lines 60-83
export async function getUserInfo(): Promise<UserInfo | null> {
  try {
    const response = await fetch("/.auth/me");
    if (!response.ok) {
      return null;
    }
    const principals = (await response.json()) as AuthMeResponse[];
    const principal = principals[0];
    if (!principal) {
      return null;
    }
    const userId = principal.user_claims.find(
      (claim) => claim.typ === OBJECT_ID_CLAIM,
    )?.val;
    if (!userId) {
      return null;
    }
    return { userId, claims: principal.user_claims };
  } catch {
    return null;
  }
}
```

- Fetches `/.auth/me` on the **SPA's own origin** (App Service Easy Auth), not the backend.
- Narrows to the Entra **object-identifier (`oid`)** claim — the stable per-user id.
- Degrades to `null` on any of: fetch fails, `/.auth/me` non-OK, empty principal list, or missing `oid`. `null` → caller falls back to the default user.
- **No MSAL**: there is no `@azure/msal-*` import, no `useMsal`, no token acquisition. `grep` for `msal` returns nothing in `src/`.

### Default GUID — a FIXED all-zeros sentinel

`v2/src/frontend/src/api/auth.tsx` lines 35-40:

```tsx
/**
 * The all-zeros id forwarded when no signed-in user has been resolved.
 * The backend treats it as a single shared partition for local /
 * unauthenticated use.
 */
export const DEFAULT_USER_ID = "00000000-0000-0000-0000-000000000000";
```

Resolved-id singleton + getter (lines 47-88):

```tsx
let currentUserId: string | null = null;

export function getUserId(): string {
  return currentUserId ?? DEFAULT_USER_ID;
}

export function setUserId(userId: string | null): void {
  currentUserId = userId;
}
```

**Where a default GUID would be generated: nowhere.** The default is a hard-coded
all-zeros constant, not a runtime-generated GUID. `crypto.randomUUID()` is used
in the FE only for unrelated ids:

- `v2/src/frontend/src/pages/chat/components/MessageInput.tsx` line 57 — chat message id.
- `v2/src/frontend/src/pages/admin/IngestData/IngestData.tsx` line 221 — ingest batch id.

Neither feeds `user_id`.

### Initials — `v2/src/frontend/src/components/Header/userIdentity.tsx`

Computed from the resolved display name (see Q4 for the full function).

### `/.auth/me` vs MSAL

**Easy Auth `/.auth/me`** is the only mechanism. No MSAL flow exists.

---

## Q3 — Auth enable/disable flag

**There is no explicit frontend auth toggle.** Findings:

- `grep` across `v2/src/frontend/**` for `VITE_AUTH`, `ENABLE_AUTH`, `enableAuth`,
  `authEnabled` → **no source matches** (only a stray substring inside the built
  bundle `dist/assets/index-*.js`, which is minified vendor code, not FE config).
- The **only** `VITE_*` variable the FE reads is `VITE_BACKEND_URL`
  (`v2/src/frontend/src/api/runtimeConfig.tsx` line 39):

  ```tsx
  return (import.meta.env.VITE_BACKEND_URL as string | undefined) ?? "";
  ```

  `vite.config.ts` line 9 documents it as the backend origin, not an auth switch.

- Auth is **implicit / capability-detected**: the bootstrap calls
  `getUserInfo()`; if `/.auth/me` yields a principal, the real `oid` is used;
  otherwise the default user is used. "Auth disabled" is simply the state where
  `/.auth/me` returns nothing (no identity provider bound to the App Service).

**Conclusion:** the intended "auth ENABLED vs DISABLED" flag does **not** exist as
a discrete frontend config value. The behavior is driven entirely by the presence
or absence of an Easy Auth principal at runtime.

---

## Q4 — Current user display / initials + the `G` fallback

### Display-name + initials helpers — `v2/src/frontend/src/components/Header/userIdentity.tsx`

Guest constant + display-name resolver (lines 14-48):

```tsx
const GUEST_NAME = "Guest";

const DISPLAY_NAME_CLAIM_TYPES = [
  "name",
  "preferred_username",
  "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
  "emails",
];

export function resolveDisplayName(
  userInfo: UserInfo | null | undefined,
): string {
  if (userInfo === null || userInfo === undefined) {
    return GUEST_NAME;
  }
  for (const claimType of DISPLAY_NAME_CLAIM_TYPES) {
    const value = userInfo.claims
      .find((claim) => claim.typ === claimType)
      ?.val.trim();
    if (value !== undefined && value !== "") {
      return value;
    }
  }
  return GUEST_NAME;
}
```

Initials derivation (lines 50-69):

```tsx
export function userInitials(name: string): string {
  const cleaned = name.replace(/\s*\([^)]*\)/g, "").trim();
  if (cleaned === "") {
    return "G";
  }
  const parts = cleaned.split(/\s+/).filter(Boolean);
  if (parts.length >= 2) {
    const first = parts[0]?.charAt(0) ?? "";
    const second = parts[1]?.charAt(0) ?? "";
    return (first + second).toUpperCase();
  }
  return cleaned.charAt(0).toUpperCase();
}
```

**How `G` is produced:** two independent paths both yield `G`:

1. No resolved user → `resolveDisplayName(null)` → `"Guest"` →
   `userInitials("Guest")` → single-part name → `"Guest".charAt(0).toUpperCase()`
   → `"G"`.
2. Empty name → `userInitials("")` → `cleaned === ""` guard → literal `"G"`.

So the "default → G" behavior is intentional and works.

### The avatar component — `v2/src/frontend/src/components/Header/HeaderTools.tsx`

Lines ~60-62 resolve the name; lines ~113-121 render the Fluent `<Avatar>`:

```tsx
  const displayName = resolveDisplayName(userInfo);
```

```tsx
      <Avatar
        shape="circular"
        color="neutral"
        name={displayName}
        initials={userInitials(displayName)}
        size={28}
        title={displayName}
        data-testid="header-user-avatar"
      />
```

`userInfo` flows in from `App.tsx` → `<Header userInfo={auth.userInfo} />` →
`<HeaderTools userInfo={userInfo} />`. When unauthenticated, `auth.userInfo` is
`null`, so the avatar shows `G` with a `Guest` title.

---

## Q5 — Models / types + header-vs-body

### `v2/src/frontend/src/models/auth.tsx`

```tsx
// External Easy Auth /.auth/me wire shape (snake_case, platform-owned)
export interface UserClaim {
  typ: string;
  val: string;
}

export interface AuthMeResponse {
  user_id: string;
  user_claims: UserClaim[];
  provider_name: string;
}

// FE-owned domain shapes
export interface UserInfo {
  userId: string;
  claims: UserClaim[];
}

export const AuthPhase = {
  Loading: "loading",
  Resolved: "resolved",
} as const;
export type AuthPhase = (typeof AuthPhase)[keyof typeof AuthPhase];

export interface AuthState {
  userId: string;
  userInfo: UserInfo | null;
  phase: AuthPhase;
}
```

Note: `AuthMeResponse.user_id` is the external Easy Auth payload's email/UPN
field — the FE deliberately **prefers the `oid` claim** over it (see Q2). The
FE-owned `UserInfo.userId` holds the `oid`.

### Auth state machine — `v2/src/frontend/src/hooks/useAuth.tsx`

```tsx
const INITIAL_AUTH_STATE: AuthState = {
  userId: DEFAULT_USER_ID,
  userInfo: null,
  phase: AuthPhase.Loading,
};

// resolve(userInfo):
  const resolve = useCallback((userInfo: UserInfo | null) => {
    if (userInfo) {
      setUserId(userInfo.userId);        // sync the api/auth.tsx singleton
      setAuth({ userId: userInfo.userId, userInfo, phase: AuthPhase.Resolved });
      return;
    }
    setUserId(null);                     // clear override → DEFAULT_USER_ID
    setAuth({ userId: DEFAULT_USER_ID, userInfo: null, phase: AuthPhase.Resolved });
  }, []);
```

The React auth state and the module-level singleton in `api/auth.tsx` are kept in
sync via `setUserId()`. Header forwarding reads the singleton (synchronous), not
React context.

### Bootstrap wiring — `v2/src/frontend/src/App.tsx`

Import (line 47) + mount effect (lines ~110-140) resolve identity right after the
health probe:

```tsx
import { getUserInfo } from "./api/auth";
import { useAuth } from "./hooks/useAuth";
// ...
  const { auth, resolve } = useAuth();
// ...
        const userInfo = await getUserInfo();
        if (!cancelled) {
          resolve(userInfo);
        }
```

### Chat request body — `v2/src/frontend/src/models/chat.tsx` + `streamChat.tsx`

`StreamMessage` (models/chat.tsx lines 26-29):

```tsx
export interface StreamMessage {
  role: "user" | "assistant" | "system";
  content: string;
}
```

Request body assembled in `streamChat.tsx` (`streamChatOnce`, ~lines 232-236):

```tsx
  const payload =
    conversationId !== null
      ? { messages, conversation_id: conversationId }
      : { messages };
```

**Conclusion:** `user_id` is **not** part of the request body. The body is
`{ messages, conversation_id? }`. The user id travels **only** as the
`x-ms-client-principal-id` HTTP header.

---

## Evidence index (file : line : what)

- `v2/src/frontend/src/api/auth.tsx:33` — `PRINCIPAL_ID_HEADER = "x-ms-client-principal-id"`.
- `v2/src/frontend/src/api/auth.tsx:40` — `DEFAULT_USER_ID = "00000000-0000-0000-0000-000000000000"` (fixed sentinel).
- `v2/src/frontend/src/api/auth.tsx:60-83` — `getUserInfo()` reads `/.auth/me`, narrows to `oid`.
- `v2/src/frontend/src/api/auth.tsx:86-88` — `getUserId()` returns resolved id or `DEFAULT_USER_ID`.
- `v2/src/frontend/src/api/auth.tsx:~100-107` — `userIdHeaders()` builds the header.
- `v2/src/frontend/src/api/streamChat.tsx:39,244-250` — spreads `userIdHeaders()` on `POST /api/conversation`; body `{ messages, conversation_id? }`.
- `v2/src/frontend/src/api/conversationHistory.tsx:22,146` — spreads header on history GET.
- `v2/src/frontend/src/api/admin.tsx:26,120,147,175,198,223,252,277` — spreads header on all admin calls.
- `v2/src/frontend/src/api/speech.tsx:17,56` — spreads header on speech token mint.
- `v2/src/frontend/src/api/runtimeConfig.tsx:39,58` — `VITE_BACKEND_URL` fallback; `/config` fetch has NO user header.
- `v2/src/frontend/src/hooks/useAuth.tsx:22-66` — auth state machine, `DEFAULT_USER_ID` initial + `resolve()`.
- `v2/src/frontend/src/models/auth.tsx` — `UserClaim` / `AuthMeResponse` / `UserInfo` / `AuthPhase` / `AuthState`.
- `v2/src/frontend/src/components/Header/userIdentity.tsx:14-69` — `resolveDisplayName` + `userInitials` (`G` fallback).
- `v2/src/frontend/src/components/Header/HeaderTools.tsx:~60,113-121` — `<Avatar initials={userInitials(displayName)}>`.
- `v2/src/frontend/src/App.tsx:47,~110-140` — bootstrap calls `getUserInfo()` then `resolve()`.
- `v2/src/frontend/src/models/chat.tsx:26-29` — `StreamMessage` shape (no user_id).

## Recommended next research (not completed here)

- [ ] Backend side: how `POST /api/conversation` and the history routes read
      `x-ms-client-principal-id` and whether the all-zeros sentinel is accepted as
      a valid partition key (backend `v2/src/backend/**`) — needed to judge whether
      BUG-0090's fix belongs in the FE (generate a real GUID) or backend (map the
      sentinel).
- [ ] Whether the backend *also* trusts its own server-injected Easy Auth claim
      over the forgeable FE header (the auth.tsx docstring implies admin RBAC uses
      the server claim) — confirms the FE header is chat-history-only.
- [ ] Confirm the intended BUG-0090 semantics: does "defaults to a GUID" mean a
      per-session unique `crypto.randomUUID()` (so each anon user gets a private
      history) or is the shared all-zeros sentinel acceptable?
- [ ] Frontend tests: `v2/tests/frontend/**` coverage of `userIdHeaders`,
      `getUserInfo`, and `userInitials` — to see what behavior is pinned before any
      fix changes the default-GUID logic.

## Clarifying questions for the user

1. **Does BUG-0090 want a UNIQUE GUID per anonymous session** (via
   `crypto.randomUUID()`, persisted in e.g. `localStorage`), or is the current
   **shared all-zeros sentinel** the intended "default GUID"? This is the single
   biggest behavioral gap and changes where the fix lands.
2. **Is an explicit FE auth on/off flag actually desired**, or is the current
   implicit `/.auth/me`-presence detection sufficient? The user's description says
   "when auth is ENABLED/DISABLED", but no such toggle exists in the FE today.
3. **Should the initial for a generated (non-Guest) anonymous GUID still be `G`**,
   or something derived from the GUID? Today any unresolved user shows `G` via the
   "Guest" name; a random GUID would still show `G` unless the naming changes.
