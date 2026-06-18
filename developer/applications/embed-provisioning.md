# Embedding Mentor AI with Backend Token Provisioning

> **Base URL:** `{DM_BASE}/api/core/`
> **Authentication:** `Authorization: Api-Token <PLATFORM_API_KEY>`

If you embed the Mentor AI widget, the standard flow authenticates the user through the
Auth SPA SSO (login popup → IDP → redirect) before the embed has the tokens it needs. This
guide shows how to **replace that SSO leg with two server-to-server calls** from your own
backend: provision the user's tokens from Data Manager (DM), then hand the embed the same
`ibl-data` blob the Auth SPA would have produced. No popup, no IDP round-trip.

Three pieces:

1. `POST /api/core/consolidated-token/provision/` — resolve-or-create the user, mint `dm_token`, `axd_token`, and an edX-signed `edx_jwt_token`.
2. `GET  /api/core/users/platforms/` — fetch the user's tenant/platform record.
3. Assemble those into the `ibl-data` payload and pass it to the iframe.

`DM_BASE` is your tenant's Data Manager host, e.g. `https://base.manager.iblai.app`.

---

## Authentication

Every call below is **server-to-server** and authenticates with a tenant-scoped
**PlatformApiKey** (created from *Edit Mentor → API tab → Create API Key*):

```
Authorization: Api-Token <PLATFORM_API_KEY>
```

This key can mint a token for *any* user on the tenant, so it must live **only on your
backend** — never in browser JS or the embed snippet. The provisioning endpoint is also
gated per-tenant by the IBL-managed flag `ENABLE_PLATFORM_CONSOLIDATED_PROXY_PROVISIONING`
(must be boolean `true`, else the endpoint returns **404**). Contact IBL to enable it for
your tenant.

---

## 1. Provision + mint tokens

```
POST {DM_BASE}/api/core/consolidated-token/provision/
Authorization: Api-Token <PLATFORM_API_KEY>
Content-Type: application/json
```

### Request

```json
{
  "username": "johndoe",
  "email": "johndoe@example.com",
  "name": "John Doe",
  "platform_key": "iblai"
}
```

- `username` **and** `email` are both required (the endpoint may need to *create* the user on edX).
- `name` is optional; used only when creating a brand-new edX user.
- `platform_key` must equal the PlatformApiKey's own platform, or the call is rejected `403`.

### Behavior

| Situation | Result |
|---|---|
| No edX user for either username/email | Created on edX (no password — non-SSO), synced to DM, linked, tokens minted → **200** |
| `username`+`email` both match the same edX user | Idempotent: linked + minted → **200** |
| `email` maps to a *different* username (or vice versa) | **403** `{"detail":"Invalid Request"}` (anti-enumeration) |
| Tenant flag not boolean `true` | **404** |
| User created on edX but not yet synced to DM | **503** + `Retry-After` header — re-POST the identical body |
| Per-user token cap exceeded | **403** |

### Success response (`200`)

```json
{
  "data": {
    "user": {
      "user_id": 1234,
      "user_email": "johndoe@example.com",
      "user_nicename": "johndoe",
      "user_display_name": "johndoe",
      "user_fullname": "John Doe"
    },
    "axd_token":      { "token": "<axd-token>",  "expires": "2026-06-03T12:00:00Z" },
    "dm_token":       { "token": "<dm-token>",   "expires": "2026-06-03T12:00:00Z" },
    "edx_jwt_token":  { "token": "<edx-jwt>",    "expires": "2026-06-03T12:00:00Z" }
  }
}
```

---

## 2. Fetch the user's platform/tenant

```
GET {DM_BASE}/api/core/users/platforms/?username=johndoe&platform_key=iblai
Authorization: Api-Token <PLATFORM_API_KEY>
```

A PlatformApiKey caller is scoped to its own platform, so this returns **exactly the one
link for that tenant**.

### Response (`200`)

```json
[
  {
    "user_id": 1234,
    "username": "johndoe",
    "email": "johndoe@example.com",
    "key": "iblai",
    "org": "iblai",
    "platform_name": "IBL AI",
    "lms_url": "https://learn.iblai.app",
    "cms_url": "https://studio.learn.iblai.app",
    "is_admin": false,
    "is_staff": false,
    "active": true
  }
]
```

The fields used downstream are `key`, `org`, `is_admin`, and `username`.

---

## 3. Formulate `ibl-data`

The embed hydrates its `localStorage` from a single `ibl-data` JSON object — the same object
the Auth SPA builds after SSO. Map the two responses above onto its keys. Two non-obvious
details to match:

- **`current_tenant` is reduced to just `{ "key": ... }`** — not the full tenant object.
- **`tenants` carries only `{ key, name, is_admin, username }`** per tenant, where `name`
  is the platform's `org`.

Token, expiry, and `userData` values are passed through as-is. `current_tenant`, `tenants`,
and `userData` are **JSON strings** (stringify them before embedding).

```jsonc
{
  // tokens (from /provision data — raw token strings, not the {token,expires} objects)
  "axd_token":          "<axd-token>",
  "axd_token_expires":  "2026-06-03T12:00:00Z",
  "dm_token":           "<dm-token>",
  "dm_token_expires":   "2026-06-03T12:00:00Z",

  // edX-signed JWT (from /provision data.edx_jwt_token); the SPA forwards it for edX-backed calls
  "edx_jwt_token":         "<edx-jwt>",
  "edx_jwt_token_expires": "2026-06-03T12:00:00Z",

  // identity (from /provision data.user) — JSON string
  "userData": "{\"user_id\":1234,\"user_nicename\":\"johndoe\",\"user_email\":\"johndoe@example.com\",\"user_fullname\":\"John Doe\"}",

  // tenant (from /users/platforms) — JSON strings; note reduced shapes
  "tenant": "iblai",
  "current_tenant": "{\"key\":\"iblai\"}",
  "tenants": "[{\"key\":\"iblai\",\"name\":\"iblai\",\"is_admin\":false,\"username\":\"johndoe\"}]"
}
```

Field mapping from the two DM calls:

| `ibl-data` key | Source |
|---|---|
| `axd_token` / `axd_token_expires` | `/provision` → `data.axd_token.{token,expires}` |
| `dm_token` / `dm_token_expires` | `/provision` → `data.dm_token.{token,expires}` |
| `userData` | `JSON.stringify(/provision data.user)` |
| `tenant` | `/users/platforms` → `key` |
| `current_tenant` | `JSON.stringify({ key })` from `/users/platforms` → `key` |
| `tenants` | `JSON.stringify([{ key, name: org, is_admin, username }])` from `/users/platforms` |
| `edx_jwt_token` / `edx_jwt_token_expires` | `/provision` → `data.edx_jwt_token.{token,expires}` |

Build this object **on your backend** (it carries the freshly minted tokens) and return it
to the page that hosts the widget.

---

## 4. Hand `ibl-data` to the iframe

The embed URL is the standard one:

```
{mentorIframeUrl}/platform/{tenant}/{mentorId}?embed=true
```

Pick one delivery path:

**Option A — `ibl-data` query param.** Append the URL-encoded JSON to the iframe `src`. The
embed reads it, seeds `localStorage`, then strips the param. This mirrors what the Auth SPA
does on its redirect.

```js
const src = `${mentorIframeUrl}/platform/${tenant}/${mentorId}`
          + `?embed=true&ibl-data=${encodeURIComponent(JSON.stringify(iblData))}`;
iframe.src = src;
```

**Option B — `postMessage`.** Load the iframe first, then push the same blob once it's ready.
The embed listens for `MENTOR:AUTH_UPDATE`:

```js
iframe.contentWindow.postMessage(
  { type: 'MENTOR:AUTH_UPDATE', authData: JSON.stringify(iblData) },
  '*'
);
```

Either path ends with the embed writing the tokens + tenant into `localStorage` and
rendering as the authenticated user.
</content>
