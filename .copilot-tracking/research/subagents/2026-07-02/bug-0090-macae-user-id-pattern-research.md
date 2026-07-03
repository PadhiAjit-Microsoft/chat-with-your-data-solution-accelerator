<!-- markdownlint-disable-file -->
# Research: MACAE backend `user_id` extraction pattern (BUG-0090)

Repo (READ-ONLY external reference): https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator

Branch inspected: `main` (as of 2026-07-02).

## Research goal

Document EXACTLY how MACAE's backend obtains `user_id` from an incoming HTTP request and how it behaves when auth is enabled vs disabled, so CWYD can mirror the contract (frontend passes a user_id header; default GUID when auth off, real principal id when auth on; backend reads the header).

## TL;DR — MACAE's exact `user_id` contract

- **Header read:** `x-ms-client-principal-id` (the signed-in user's GUID), plus `x-ms-client-principal-name`, `x-ms-client-principal-idp`, `x-ms-token-aad-id-token`, `x-ms-client-principal` (base64 principal blob).
- **Injected by:** Azure App Service / Container Apps **Easy Auth** reverse proxy when auth is ON. The MACAE **frontend also explicitly sets** `x-ms-client-principal-id` on every request via an httpClient interceptor.
- **Fallback when header absent:** the helper imports a `sample_user` dict and returns the sample principal id `"00000000-0000-0000-0000-000000000000"` (all-zero GUID). **This is a silent dev-mode fallback — the helper itself NEVER raises 401.**
- **401 behavior:** the extraction helper never 401s. Individual v4 data endpoints defensively do `if not user_id: raise HTTPException(401, ...)`, but because the helper always returns the all-zero GUID when the header is missing, that guard only fires when the header is present-but-empty — not on a plain missing header in dev mode.
- **GUID validation:** **None.** The helper passes the header string through verbatim. Routers only test truthiness (`if not user_id`). No `uuid` parse, no regex, no format check anywhere.
- **Frontend default:** `getUserId()` returns `USER_ID ?? "00000000-0000-0000-0000-000000000000"` — the same all-zero GUID default when there's no signed-in user. This is exactly the "default GUID when auth off, real principal id when auth on" pattern CWYD wants.

---

## Q1 — `user_id` extraction helper (full code + headers + fallback)

**File:** `src/backend/auth/auth_utils.py`
Link: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/auth/auth_utils.py

Full function code (lines 1–32):

```python
import base64
import json
import logging


def get_authenticated_user_details(request_headers):
    user_object = {}

    # check the headers for the Principal-Id (the guid of the signed in user)
    if "x-ms-client-principal-id" not in request_headers:
        logging.info("No user principal found in headers")
        # if it's not, assume we're in development mode and return a default user
        from . import sample_user

        raw_user_object = sample_user.sample_user
    else:
        # if it is, get the user details from the EasyAuth headers
        raw_user_object = {k: v for k, v in request_headers.items()}

    normalized_headers = {k.lower(): v for k, v in raw_user_object.items()}
    user_object["user_principal_id"] = normalized_headers.get(
        "x-ms-client-principal-id"
    )
    user_object["user_name"] = normalized_headers.get("x-ms-client-principal-name")
    user_object["auth_provider"] = normalized_headers.get("x-ms-client-principal-idp")
    user_object["auth_token"] = normalized_headers.get("x-ms-token-aad-id-token")
    user_object["client_principal_b64"] = normalized_headers.get(
        "x-ms-client-principal"
    )
    user_object["aad_id_token"] = normalized_headers.get("x-ms-token-aad-id-token")

    return user_object
```

There is a second helper in the same file, `get_tenantid(client_principal_b64)` (lines 34–49), which base64-decodes the `x-ms-client-principal` blob and reads the `tid` (tenant id) claim; on any decode error it logs and returns `""`. Not directly relevant to `user_id`, but shows MACAE's "decode-and-tolerate" posture.

### Headers read (all lower-cased before lookup)

| Output key            | Source header                 | Meaning                          |
| --------------------- | ----------------------------- | -------------------------------- |
| `user_principal_id`   | `x-ms-client-principal-id`    | signed-in user GUID (the user_id)|
| `user_name`           | `x-ms-client-principal-name`  | UPN / display name               |
| `auth_provider`       | `x-ms-client-principal-idp`   | e.g. `aad`                       |
| `auth_token`          | `x-ms-token-aad-id-token`     | AAD id token                     |
| `client_principal_b64`| `x-ms-client-principal`       | base64 principal claims blob     |
| `aad_id_token`        | `x-ms-token-aad-id-token`     | same as `auth_token`             |

Header lookup is **case-insensitive**: the helper rebuilds `normalized_headers = {k.lower(): v ...}` before `.get(...)`. (Confirmed by `test_with_mixed_case_headers`.)

### Fallback value when `x-ms-client-principal-id` is absent

**File:** `src/backend/auth/sample_user.py`
Link: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/auth/sample_user.py

Relevant keys of the `sample_user` dict:

```python
sample_user = {
    ...
    "X-Ms-Client-Principal": "your_base_64_encoded_token",
    "X-Ms-Client-Principal-Id": "00000000-0000-0000-0000-000000000000",
    "X-Ms-Client-Principal-Idp": "aad",
    "X-Ms-Client-Principal-Name": "testusername@constoso.com",
    "X-Ms-Token-Aad-Id-Token": "your_aad_id_token",
    ...
}
```

So the **fallback `user_id` is the all-zero GUID `"00000000-0000-0000-0000-000000000000"`** and the fallback name is `"testusername@constoso.com"` (note MACAE's own typo "constoso"). Verified by `src/backend/tests/auth/test_sample_user.py`:

```python
assert (
    sample_user["X-Ms-Client-Principal-Id"]
    == "00000000-0000-0000-0000-000000000000"
)
assert sample_user["X-Ms-Client-Principal-Name"] == "testusername@constoso.com"
```

---

## Q2 — Does MACAE 401 when the header is missing?

**The extraction helper does NOT 401.** When `x-ms-client-principal-id` is absent it logs `"No user principal found in headers"` and silently substitutes the `sample_user` (all-zero GUID). This is the "assume development mode" branch. There is no `raise` in `auth_utils.py`.

**Individual v4 data endpoints add a defensive truthiness guard**, e.g. `get_team_config_by_id` in `src/backend/v4/api/router.py`:
Link: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/v4/api/router.py

```python
@app_v4.get("/team_configs/{team_id}")
async def get_team_config_by_id(team_id: str, request: Request):
    # Validate user authentication
    authenticated_user = get_authenticated_user_details(request_headers=request.headers)
    user_id = authenticated_user["user_principal_id"]
    if not user_id:
        raise HTTPException(
            status_code=401, detail="Missing or invalid user information"
        )
    ...
```

Key nuance: because the helper always returns the all-zero GUID when the header is completely missing, `user_id` is truthy in that case, so **this 401 does NOT fire on a plain missing header** — it only fires if the header is present but empty-string (`x-ms-client-principal-id: ""`), which the sample-user branch is skipped for (the key *is* present, just empty). In normal auth-off dev flow, requests succeed with the zero GUID; the 401 is a defensive edge guard, not an auth gate.

Net: **MACAE silently falls back to a default user; it does not hard-require the header at the helper level.** Auth enforcement (when enabled) is done upstream by Easy Auth / the reverse proxy, not by this Python code.

---

## Q3 — GUID validation

**No.** MACAE does not validate that `user_id` is a GUID anywhere in the request path:

- `get_authenticated_user_details` returns the header value verbatim (`normalized_headers.get("x-ms-client-principal-id")`).
- Routers only check truthiness (`if not user_id`).
- Tests explicitly pass non-GUID values through and expect them unchanged, e.g. `test_error_resilience_complete_flow`:
  ```python
  malformed_headers = {"x-ms-client-principal-id": "malformed-id", ...}
  user_details = get_authenticated_user_details(malformed_headers)
  assert user_details["user_principal_id"] == "malformed-id"
  ```
  and `test_with_partial_auth_headers` uses `"partial-test-id"`.

There is **no `uuid.UUID(...)` parse, no regex, no format assertion** on `user_id`.

---

## Q4 — How the frontend sends `user_id`

MACAE uses **both** mechanisms: the frontend explicitly sets the header AND Easy Auth injects the `x-ms-client-principal-*` headers at the proxy.

### Frontend explicitly sets the header (httpClient request interceptor)

**File:** `src/App/src/api/httpClient.ts`
Link: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/App/src/api/httpClient.ts

```ts
import { getUserId } from './config';
...
httpClient.addRequestInterceptor((config) => {
    const userId = getUserId();
    const token = localStorage.getItem('token');

    const headers = new Headers(config.headers as HeadersInit);
    ...
    if (userId) {
        headers.set('x-ms-client-principal-id', String(userId));
    }
    if (token) {
        headers.set('Authorization', `Bearer ${token}`);
    }

    return { ...config, headers };
});
```

The interceptor docstring calls itself the "single source of truth for userId header."

### Frontend `getUserId()` — default GUID when no user

**File:** `src/App/src/api/config.tsx`
Link: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/App/src/api/config.tsx

```ts
export function getUserId(): string {
    if (!USER_ID) {
        USER_ID = getUserInfoGlobal()?.user_id || null;
    }
    const userId = USER_ID ?? "00000000-0000-0000-0000-000000000000";
    return userId;
}
```

So the frontend **defaults to the same all-zero GUID `"00000000-0000-0000-0000-000000000000"`** when there is no signed-in user — mirroring the backend `sample_user` fallback. This is precisely the CWYD-desired "default GUID when auth off, real principal id when auth on."

### Where the real `user_id` comes from (auth on)

`getUserInfo()` fetches Easy Auth's `/.auth/me` and reads the AAD object-id claim:

```ts
export async function getUserInfo(): Promise<UserInfo> {
    const response = await fetch("/.auth/me");
    ...
    const userInfo: UserInfo = {
        ...
        user_id: payload[0].user_claims?.find(
            (claim) => claim.typ === 'http://schemas.microsoft.com/identity/claims/objectidentifier'
        )?.val || '',
    };
    return userInfo;
}
```

### Auth toggle plumbing

- Frontend server `src/App/frontend_server.py` exposes `GET /config` returning `{"API_URL": ..., "ENABLE_AUTH": os.getenv("AUTH_ENABLED", "false")}`.
- `src/App/src/index.tsx` fetches `/config`, coerces `ENABLE_AUTH` via `toBoolean`, then calls `getUserInfo()` to populate the global user info.
- Default config in `config.tsx` is `ENABLE_AUTH: false` — auth OFF by default (dev-first), matching the CWYD config-defaults preference.

---

## Q5 — Admin/status/health endpoints and auth gating

No dedicated "admin-status" endpoint that hard-requires the principal header was found. Health endpoints are **not** user-auth gated:

- **Backend health middleware** `src/backend/middleware/health_check.py`:
  Link: https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/middleware/health_check.py
  ```python
  class HealthCheckMiddleware(BaseHTTPMiddleware):
      async def dispatch(self, request, call_next):
          if request.url.path == self.__healthz_path:
              status = await self.check()
              status_code = 200 if status.status else 503
              status_message = "OK" if status.status else "Service Unavailable"
              if (self.password is not None
                      and request.query_params.get("code") == self.password):
                  return JSONResponse(jsonable_encoder(status), status_code=status_code)
              return PlainTextResponse(status_message, status_code=status_code)
          response = await call_next(request)
          return response
  ```
  Gated by an **optional `?code=<password>` query param** (to reveal detailed JSON), **not** by `x-ms-client-principal-*`. Plain `/healthz` returns text OK/503 with no auth.

- **Frontend server** `src/App/frontend_server.py`: `GET /health` → `{"status": "healthy"}` and `GET /config` are both unauthenticated.

The only endpoints that "hard-require" a user are the **v4 data endpoints** (e.g. `/team_configs/{team_id}`) that read `user_principal_id` and raise 401 on a falsy value — and as noted in Q2, that guard is effectively bypassed by the all-zero-GUID fallback under auth-off dev flow.

---

## Source file index (all GitHub links)

- Backend helper: `src/backend/auth/auth_utils.py`
  https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/auth/auth_utils.py
- Dev fallback dict: `src/backend/auth/sample_user.py`
  https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/auth/sample_user.py
- v4 router 401 guard: `src/backend/v4/api/router.py`
  https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/v4/api/router.py
- Health middleware: `src/backend/middleware/health_check.py`
  https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/backend/middleware/health_check.py
- Frontend httpClient interceptor: `src/App/src/api/httpClient.ts`
  https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/App/src/api/httpClient.ts
- Frontend `getUserId` / `getUserInfo`: `src/App/src/api/config.tsx`
  https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/App/src/api/config.tsx
- Frontend server `/config` + `/health`: `src/App/frontend_server.py`
  https://github.com/microsoft/Multi-Agent-Custom-Automation-Engine-Solution-Accelerator/blob/main/src/App/frontend_server.py
- Tests (contract confirmation): `src/backend/tests/auth/test_auth_utils.py`, `src/tests/backend/auth/test_auth_utils.py`, `src/backend/tests/auth/test_sample_user.py`

---

## Contract summary for CWYD to mirror

1. Backend reads a single header, `x-ms-client-principal-id`, case-insensitively, as the `user_id`.
2. If absent → silent dev fallback to the all-zero GUID `00000000-0000-0000-0000-000000000000` (no 401 at the helper).
3. No GUID/format validation — the string is passed through verbatim.
4. Frontend explicitly sets `x-ms-client-principal-id` on every request via a request interceptor; `getUserId()` defaults to the same all-zero GUID when no user is signed in; the real principal id comes from Easy Auth `/.auth/me` (`objectidentifier` claim). Easy Auth also injects `x-ms-client-principal-*` at the proxy when auth is on.
5. Health/config endpoints are not user-auth gated; only v4 data endpoints add a defensive `if not user_id: 401` that the fallback effectively neutralizes in dev mode.

## Clarifying questions

1. CWYD BUG-0090 target: do you want CWYD to replicate MACAE's **silent fallback (never 401)** exactly, or a stricter variant where the backend **does 401** when the header is missing *and* `AZURE_ENVIRONMENT=production` (i.e. fallback only in dev)? MACAE itself never 401s at the helper regardless of environment.
2. Should CWYD **validate** the incoming `user_id` as a real GUID (MACAE does not) — e.g. reject non-GUID values at the boundary per CWYD Hard Rule #14 SDK-boundary resilience — or match MACAE's pass-through behavior?
3. CWYD's chosen fallback constant: reuse MACAE's `00000000-0000-0000-0000-000000000000`, or a CWYD-specific default GUID? (Affects both the frontend default and any backend dev fallback.)
