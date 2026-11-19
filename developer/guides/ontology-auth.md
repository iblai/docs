# Authentication: the ibl.ai session, exactly as iblai/os does it

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/auth.md`](https://github.com/iblai/ontology/blob/main/docs/auth.md). This page is generated — edit it in the repository, not here.

The console issues no tokens and keeps no sessions. It reuses the ibl.ai sign-in that os.ibl.ai
uses, with the same storage keys, the same header schemes and the same refresh path, so a person
signed in to the OS is signed in here. This is the model, taken from `github.com/iblai/os`
(`providers/index.tsx`, `features/utils.ts`, `lib/token-refresh.ts`, `lib/auth-storage.ts`,
`app/sso-login/page.tsx`).

## 1. Sign-in

1. An unauthenticated visit redirects to the auth app:
   `https://<auth host>/login?app=custom&redirect-to=<this origin>`. The console pins no platform,
   so the auth app signs the person in to their own. The auth app is `login.<domain>` (hosted) or
   whatever `NEXT_PUBLIC_AUTH_URL` points at.
2. The auth app authenticates the person with the institution's SSO (Entra, Google, Shibboleth,
   whatever the platform configured on ibl.ai) and, through ibl.ai's consolidated-token proxy, mints
   the **app token pair** for that platform.
3. It returns to this origin at `/sso-login` (the OS route; the Vibe starter calls it
   `/sso-login-complete`) carrying the session. The page is the SDK's `SsoLogin` component; it writes
   the session to storage and continues to the saved `redirect-to` path.

## 2. What is stored, and where

The SDK's per-tab auth storage, mirrored by the app (`lib/auth-storage.ts`):

| Key                                                      | Content                                                                                                                          | Lifetime                               |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `axd_token`, `axd_token_expires`                         | token for the analytics/AXD service                                                                                              | short-lived; expiry stored as ISO date |
| `dm_token`, `dm_token_expires`                           | token for the data-manager service (`…/dm/api/…`, which is where MCP servers, RBAC, api-tokens, org metadata and analytics live) | short-lived                            |
| `edx_jwt_token`                                          | LMS JWT; **not used by this console** (no LMS calls)                                                                             | —                                      |
| `userData`, `user_metadata`                              | the signed-in user and their metadata                                                                                            | session                                |
| `current_tenant`, `tenant`, `tenants`, `visiting_tenant` | the active platform (`current_tenant`, an object; `tenant`, its key), memberships                                                | session                                |

Default is plain `localStorage`. With `NEXT_PUBLIC_IBL_PER_TAB_AUTH=true` the SDK keeps
`sessionStorage` as the per-tab source of truth and `localStorage` as the most-recent-login seed, so
two tabs can sit in two platforms. We keep the default.

Tokens never leave the browser except to ibl.ai and to the console's route handlers over TLS, which
run on the platform's own server. On a console hosted elsewhere (the pages-only build) those
handlers forward the `dm_token` to the platform's own server, so it reaches two places besides
ibl.ai: that console, in passing, and the server. Neither stores it.

## 3. Which header goes to which service

From `features/utils.ts` in the OS:

| Service                             | Header                               |
| ----------------------------------- | ------------------------------------ |
| DM (`config.dmUrl()`, `…/dm/api/…`) | `Authorization: Token <dm_token>`    |
| AXD (`config.axdUrl()`)             | `Authorization: Token <axd_token>`   |
| LMS (`config.lmsUrl()`)             | `Authorization: JWT <edx_jwt_token>` |
| anything else                       | `Authorization: Token <dm_token>`    |

Two things follow for this console:

- Every ibl.ai call the console makes (MCP servers, API keys, RBAC, agents) is a DM call, so the
  browser sends `Token <dm_token>` to `https://ext.ibl.ai/dm/api/…`.
- The scheme is `Token`, not `Bearer`. The DM API answers `Bearer` with `401`. Platform API keys
  (the long-lived ones created under **API** or by the console) use a third scheme, `Api-Token`, and
  are what agents and CLIs present to the gateway. The DM endpoints that manage API keys accept
  only a session `Token`, never an `Api-Token`.

## 4. Expiry, refresh, 401

- Before using a token the SDK checks `*_token_expires`. `hasNonExpiredAuthToken()` gates the first
  render.
- On expiry, or on any `401` from ibl.ai, the OS sends the browser to the auth app's
  `/login/complete?redirect-to=<origin>&redirect-path=<current path>&tenant=<key>`
  (`lib/token-refresh.ts`). The auth app re-mints the pair from its own first-party session and
  returns the person to the same page with the session intact. If that session is gone it falls
  through to `/login?logout=1` with the same return address: one sign-in, back where they were.
- Embedded in an iframe (the OS inside another app) it instead posts `authExpired` to the host and
  waits for a fresh payload (`axd_token`, `dm_token`, `tenant`), then replays the failed request.
  The console is not embedded, so it uses the redirect path.
- Logout: clear storage, redirect to `<auth>/logout?redirect-to=<origin>&tenant=<key>`.

## 5. Platform

`current_tenant` selects the platform; the console keeps no platform of its own. Every DM call
carries `platform_key=<platform key>` (query or body). The platform menu in our header lists the
platforms the person belongs to and is the SDK's switch; switching re-mints tokens for the new
platform through the same `/login/complete` path. Every call to the host carries the same
platform as `x-ontology-platform: <platform key>`, and every check below is made against the
platform the call names. One ontology per platform follows from this: the host keeps each
platform's files apart and gives each platform its own engine ([network.md](/developer/guides/ontology-network)). On a
console hosted elsewhere (the pages-only build) each platform has its own server: the browser keeps
its address per platform and names it in `x-ontology-server` beside the platform.

## 6. How this host checks a caller

The host holds no ibl.ai secret. On every call it asks ibl.ai, with the caller's own credential
and `iblai-org: <platform>`, what that credential is granted under `/custom/ontology/` on that
platform, and who it belongs to. ibl.ai refuses unless the credential belongs to that platform.
The platform is `x-ontology-platform` for the route handlers and the first part of the address for
the gateway (the part after `/iblai` on the server's second address). The answers are kept in
memory, keyed by a hash of the credential and the platform, and never written to disk: what the
credential is granted for 60 seconds, who it belongs to and whether they administer the platform
for 300 seconds. Revoking a key or removing a policy on ibl.ai takes effect within a minute;
removing an administrator within five.

A server with an engine of its own (`AO_URL`) serves one platform, the one its `ONTOLOGY_PLATFORM`
setting names, and checks that before it asks ibl.ai anything: a host call or a gateway call for
another platform answers `403` “This server serves another platform.” A server installed as
[install.md](/developer/guides/ontology-install) says always has an engine of its own, and its operator names the platform
in its `.env`. With `AO_URL` and no `ONTOLOGY_PLATFORM` the server refuses to
start, and every host call and gateway call answers `503` naming the setting. A server with
neither serves any platform, each through the `engine.json` its operator writes in the platform's
folder; a platform with none answers `503` “No engine is connected” ([network.md](/developer/guides/ontology-network)).

| Presented                            | Accepted by                               | Checked as                                                                                               |
| ------------------------------------ | ----------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `Authorization: Token <dm_token>`    | the console's route handlers, the gateway | a session                                                                                                |
| `Authorization: Api-Token <key>`     | the gateway                               | a platform API key                                                                                       |
| `Authorization: Bearer <credential>` | the gateway                               | tried as a session, then as an API key                                                                   |
| an OIDC ID token (Entra, Google, …)  | the gateway, beside one of the above      | not checked by the gateway; passed to the runtime, which verifies issuer, audience, signature and expiry |

- **Route handlers** (`/api/host/*`) take a session `Token` only, with `x-ontology-platform`, and
  only from an admin of that platform or a holder of the console grant there
  (`/custom/ontology/console/`). A session of another platform is refused. `GET /api/host/health`
  takes no sign-in and answers `{ "ok": true }`.
- **The gateway** (`/<platform>/<slug>/mcp`, and under `/iblai` on the server's second address)
  answers `401` without a usable credential and `403` when the credential reaches no toolset. What
  a grant reaches is in [guides/permissions.md](/developer/guides/ontology-permissions).
- When ibl.ai does not answer (a 5xx or a 429), the host answers `503` and keeps nothing. Any other
  refusal from ibl.ai is passed on in ibl.ai's own words.

## 7. As implemented in this console

Sign-in is wired with the iblai/base starter's provider chain (`/iblai-vibe-auth`), on the
platform of the sign-in:

| File                                                         | Role                                                                                                                                                                             |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `providers/iblai-providers.tsx`                              | `initializeDataLayer` → Redux store → `AuthProvider` → `TenantProvider`. Redirects to the auth app when there is no valid session; persists re-minted tokens on platform switch. |
| `app/(auth)/sso-login-complete/page.tsx`                     | The SDK `SsoLogin` callback. Lives outside the providers so it can store the tokens before `AuthProvider` checks them.                                                           |
| `lib/iblai/config.ts`                                        | Service URLs. Defaults to the hosted platform; `NEXT_PUBLIC_API_BASE_URL` / `NEXT_PUBLIC_AUTH_URL` point it elsewhere (e.g. `https://ext.ibl.ai`).                               |
| `lib/iblai/auth-utils.ts`, `tenant.ts`, `storage-service.ts` | Redirect to the auth app, logout, the platform of the sign-in (`current_tenant` / `tenant`), the storage adapter the SDK expects.                                                |
| `lib/iblai/session.tsx`                                      | Reads the stored session (user, email, tenants, `is_admin`) for the shell. Holds the User/Admin switch state. Stores nothing.                                                    |
| `store/iblai-store.ts`                                       | The SDK's RTK Query slices.                                                                                                                                                      |

Configuration (`.env.local`, not committed; template in `.env.example`):

```bash
# NEXT_PUBLIC_API_BASE_URL=https://ext.ibl.ai  # closed deployment
```

No setting names the platform of the sign-in: it is the one the person signs in to. A server with an
engine of its own still serves only the platform its `ONTOLOGY_PLATFORM` names (section 6).

Verified on 2026-11-16, while the platform was still set in the environment: an unauthenticated
visit to `http://localhost:3000` redirected to
`https://login.iblai.app/login?app=custom&redirect-to=http://localhost:3000&tenant=<platform key>`.
Sign-in without a platform has not been run yet. After sign-in the auth app returns to
`/sso-login-complete`. **Every origin the console's pages are served from (`http://localhost:3000`,
`https://<host>` on the platform's own server, a console hosted elsewhere) must be on the platform's
allowed redirect origins on ibl.ai**, or the auth app will not return to it. The platform's own
server serves the pages, so its `https://<host>` is one of them.

Access rules in the shell:

- Signed in but not an admin of the platform → an "Administrators only" page. The console
  manages data sources and servers; members have nothing to do here.
- Admins get the OS User/Admin switch. In User mode the admin items in the sidebar footer
  (Invites, Management, API, Secrets, Advanced) and the Billing icon are hidden, as in the OS.

Demo mode: `NEXT_PUBLIC_DEMO=1` skips sign-in and serves a fixed "Demo Admin" session so the UI can
be reviewed and screenshotted without an account. SDK panels (Management, API, Secrets, Advanced,
Billing, Invites, Notifications) then show a sign-in notice instead of calling ibl.ai.
