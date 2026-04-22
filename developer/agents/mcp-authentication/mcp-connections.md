# MCP Server Connections

Register MCP servers, wire them to a mentor, and create authentication bindings so AI mentors can securely invoke external tools at runtime.

---

## Overview

The Model Context Protocol (MCP) lets an AI mentor call external tools — anything from a Google Drive search to a custom workflow engine — as part of a conversation. The ibl.ai platform models MCP with three objects:

| Term | Description |
| --- | --- |
| **MCP Server** | Metadata describing an external MCP endpoint (name, URL, transport, auth type, auth scope). Tenants can register any number of servers. |
| **MCP Server Connection** | An authentication binding between a tenant, mentor, or user and an MCP server. Stores the token or the reference to an OAuth-connected service. |
| **Connection Scope** | Whether the connection applies tenant-wide (`platform`), to a specific mentor (`mentor`), or to a single user (`user`). |

A working MCP integration requires three things:

1. A **registered server** record.
2. At least one **connection** providing credentials at the desired scope.
3. A **mentor** that has the MCP tool enabled and the server attached.

> **Related guides:** OAuth-backed servers need a `ConnectedService` — see [OAuth Connectors](/docs/developer/agents/mcp-authentication/oauth-connectors). Per-user OAuth servers can also authenticate learners mid-chat — see [In-Chat MCP Events](/docs/developer/agents/mcp-authentication/in-chat-mcp-events).

---

## End-to-End Flow

```
   ┌────────────────────────────────┐
   │  1. Register MCP Server        │
   │     POST /mcp-servers/         │
   └──────────────┬─────────────────┘
                  │
                  ▼
   ┌────────────────────────────────┐        ┌──────────────────────────┐
   │  2. Create connection          │◀───────│  (OAuth only)            │
   │     POST /mcp-server-          │        │  Complete OAuth flow to  │
   │         connections/           │        │  get a ConnectedService  │
   │     scope: platform/mentor/    │        └──────────────────────────┘
   │            user                │
   └──────────────┬─────────────────┘
                  │
                  ▼
   ┌────────────────────────────────┐
   │  3. Enable mcp-tool on mentor  │
   │     PUT /mentors/{id}/settings/│
   │     • tools: ["mcp-tool"]      │
   │     • mcp_servers: [9, 14]     │
   └──────────────┬─────────────────┘
                  │
                  ▼
   ┌────────────────────────────────┐
   │  4. Learner chats              │
   │     MCP tools are invoked with │
   │     runtime-resolved headers   │
   └────────────────────────────────┘
```

---

## API Summary

All endpoints live under `/api/ai-mentor/orgs/{org}/users/{user_id}/`.

| Capability | Endpoint | Method |
| --- | --- | --- |
| List servers | `/mcp-servers/` | `GET` |
| Create server | `/mcp-servers/` | `POST` |
| Update server | `/mcp-servers/{id}/` | `PUT` / `PATCH` |
| Delete server | `/mcp-servers/{id}/` | `DELETE` |
| List connections | `/mcp-server-connections/` | `GET` |
| Create connection | `/mcp-server-connections/` | `POST` |
| Update connection | `/mcp-server-connections/{id}/` | `PUT` / `PATCH` |
| Delete connection | `/mcp-server-connections/{id}/` | `DELETE` |
| Enable tool + attach servers | `/mentors/{mentor_id}/settings/` | `PUT` / `PATCH` |

Use `Authorization: Token ...` authentication. Only tenant admins may create or update servers and connections.

---

## Choose an Auth Scope

Before registering anything, decide *whose* credentials should be used. The `auth_scope` field on the server determines the resolution strategy and — critically — whether learners are prompted to connect their own account in chat.

| Pattern | `auth_scope` | Who sets it up | When to use |
| --- | --- | --- | --- |
| **Per-tenant (platform)** | `platform` | Admin creates one connection | A single API key or shared OAuth token serves the whole organization. |
| **Per-mentor** | `mentor` | Admin creates one connection per mentor | Different mentors need different access levels to the same service. |
| **Per-user (pre-provisioned)** | `user` | Admin creates a connection per user (bulk) | Credentials are known ahead of time for each learner. |
| **Per-user (in-chat OAuth)** | `user` | Each learner authenticates mid-chat | Users hold their own identity (e.g., their personal Google Drive). |

The first three patterns never prompt the user. The fourth — `auth_scope="user"` combined with `auth_type="oauth2"` — triggers the [in-chat OAuth flow](/docs/developer/agents/mcp-authentication/in-chat-mcp-events) the first time a learner uses the mentor.

---

## 1. Register a Server

The server record describes *how* to reach the MCP endpoint and *what kind* of credentials it expects.

```http
POST /api/ai-mentor/orgs/acme/users/alice/mcp-servers/ HTTP/1.1
Authorization: Token {{TOKEN}}
Content-Type: application/json

{
  "name": "Google Drive MCP",
  "description": "Search and index Drive documents",
  "url": "https://drive-mcp.example.com",
  "transport": "sse",
  "auth_type": "oauth2",
  "auth_scope": "user",
  "is_featured": false,
  "is_enabled": true
}
```

Response:

```json
{
  "id": 9,
  "platform": 42,
  "name": "Google Drive MCP",
  "description": "Search and index Drive documents",
  "url": "https://drive-mcp.example.com",
  "transport": "sse",
  "auth_type": "oauth2",
  "auth_scope": "user",
  "is_featured": false,
  "is_enabled": true,
  "created_at": "2025-11-12T12:14:50Z",
  "updated_at": "2025-11-12T12:14:50Z"
}
```

### Field reference

| Field | Values | Purpose |
| --- | --- | --- |
| `transport` | `sse`, `websocket`, `streamable_http` | Wire protocol used to talk to the MCP endpoint. |
| `auth_type` | `none`, `token`, `oauth2` | How the request is authenticated. |
| `auth_scope` | `platform` (default), `mentor`, `user` | Whose credentials are used at resolution time. |
| `is_featured` | `true` / `false` | If `true`, other tenants can create their own connections to this server. |
| `is_enabled` | `true` / `false` | Hard off-switch — disabled servers are skipped at runtime. |

### `auth_type` vs `auth_scope`

Two fields, two different questions:

| Field | Answers |
| --- | --- |
| `auth_type` | *How* is the call authenticated? (no auth, static token, or OAuth2) |
| `auth_scope` | *Whose* credentials are used? (shared platform, mentor-specific, or each user's own) |

`auth_type="oauth2"` + `auth_scope="user"` is the only combination that enables the in-chat OAuth prompt.

---

## 2. Create a Connection

A connection binds credentials to a **scope**. Start with whichever scope matches the pattern you chose above.

### Platform scope (token)

The simplest case — a single API key shared by everyone on the tenant.

**Request**

```http
POST /api/ai-mentor/orgs/acme/users/alice/mcp-server-connections/ HTTP/1.1
Authorization: Token {{TOKEN}}
Content-Type: application/json

{
  "server": 9,
  "scope": "platform",
  "auth_type": "token",
  "credentials": "super-secret-api-key",
  "authorization_scheme": "Bearer",
  "extra_headers": {
    "x-mcp-client": "mentor-ui"
  }
}
```

**Response**

```json
{
  "id": 21,
  "server": 9,
  "server_name": "Google Drive MCP",
  "scope": "platform",
  "auth_type": "token",
  "platform": 42,
  "platform_key": "acme",
  "user": null,
  "mentor": null,
  "connected_service": null,
  "connected_service_summary": null,
  "credentials": "sup****key",
  "authorization_scheme": "Bearer",
  "extra_headers": {
    "x-mcp-client": "mentor-ui"
  },
  "is_active": true,
  "created_at": "2025-11-12T12:20:00Z",
  "updated_at": "2025-11-12T12:20:00Z"
}
```

- `authorization_scheme` becomes the prefix in the `Authorization` header (e.g., `Authorization: Bearer super-secret-api-key`). Omit it to send the raw value.
- `extra_headers` is an arbitrary JSON object merged into every request.
- `credentials` is **always masked on read**; never echo the masked value back on update.

### Mentor scope (token)

Different mentors on the same tenant can present different credentials to the same server (e.g., a Finance Mentor with read/write access and a General Mentor with read-only access).

```json
{
  "server": 9,
  "scope": "mentor",
  "auth_type": "token",
  "mentor": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "credentials": "mentor-specific-key",
  "authorization_scheme": "Bearer"
}
```

- `mentor` is the mentor's `unique_id` (UUID).
- The mentor's `platform_key` must match the current tenant.
- `platform` is inferred from the mentor when omitted.

### User scope (OAuth2)

1. Ensure a [`ConnectedService`](/docs/developer/agents/mcp-authentication/oauth-connectors) exists for the target user + provider + service.
2. Reference it by ID on the connection:

```json
{
  "server": 9,
  "scope": "user",
  "auth_type": "oauth2",
  "user": "alice",
  "connected_service": 77
}
```

**Response** (abbreviated)

```json
{
  "id": 22,
  "server": 9,
  "server_name": "Google Drive MCP",
  "scope": "user",
  "auth_type": "oauth2",
  "user": "alice",
  "connected_service": 77,
  "connected_service_summary": {
    "id": 77,
    "provider": "google",
    "service": "drive",
    "user": "alice",
    "platform_key": "acme"
  },
  "credentials": "",
  "is_active": true
}
```

The platform automatically refreshes the OAuth tokens as they approach expiry — no client action required.

### Validation rules by scope

| Scope | Required fields | Forbidden fields |
| --- | --- | --- |
| `platform` | `server`, `auth_type`, (credentials or `connected_service`) | `user`, `mentor` |
| `mentor` | `server`, `auth_type`, `mentor`, (credentials or `connected_service`) | `user` |
| `user` | `server`, `auth_type`, (`user` or `connected_service`) | `mentor` |

For any `auth_type="oauth2"` connection, `connected_service` is required regardless of scope.

### Updating and deactivating

- `PATCH /mcp-server-connections/{id}/` — toggle `is_active`, swap the linked `connected_service`, rotate a token, or update headers.
- `DELETE /mcp-server-connections/{id}/` — removes the record entirely.

---

## 3. Enable the MCP Tool on a Mentor

Registering a server and its connection is only half the story. The mentor must also:

1. Have the MCP tool enabled.
2. Know which server IDs it is allowed to call.

Both are set through the mentor settings endpoint.

### Enable the tool

```http
PATCH /api/ai-mentor/orgs/acme/users/alice/mentors/{mentor_id}/settings/ HTTP/1.1
Authorization: Token {{TOKEN}}
Content-Type: application/json

{
  "tools": ["mcp-tool"]
}
```

### Attach servers to the mentor

```http
PATCH /api/ai-mentor/orgs/acme/users/alice/mentors/{mentor_id}/settings/ HTTP/1.1
Authorization: Token {{TOKEN}}
Content-Type: application/json

{
  "mcp_servers": [9, 14]
}
```

> **Replace, not merge.** The `tools` and `mcp_servers` fields are *replaced* on every update. Send the full desired list, not a delta.
>
> - `[]` clears all enabled tools / attached servers.
> - `null` leaves the existing value untouched.

Once the tool is enabled and at least one server is attached, the mentor will discover MCP tools from the attached servers and call them as part of its responses.

---

## End-to-End Example

A complete walkthrough for a tenant admin setting up a shared, tenant-wide token-based MCP server.

```bash
# 1. Register the server
curl -X POST "https://base.manager.iblai.app/api/ai-mentor/orgs/acme/users/alice/mcp-servers/" \
  -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Workflow MCP",
    "url": "https://workflow-mcp.example.com",
    "transport": "sse",
    "auth_type": "token",
    "auth_scope": "platform",
    "is_enabled": true
  }'
# → returns {"id": 9, ...}

# 2. Create a platform-scoped connection with the shared API key
curl -X POST "https://base.manager.iblai.app/api/ai-mentor/orgs/acme/users/alice/mcp-server-connections/" \
  -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "server": 9,
    "scope": "platform",
    "auth_type": "token",
    "credentials": "sk-live-abcd1234",
    "authorization_scheme": "Bearer"
  }'

# 3. Enable mcp-tool on the mentor and attach the server
curl -X PATCH "https://base.manager.iblai.app/api/ai-mentor/orgs/acme/users/alice/mentors/$MENTOR_ID/settings/" \
  -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tools": ["mcp-tool"],
    "mcp_servers": [9]
  }'
```

Any learner chatting with the mentor now transparently uses the shared Workflow MCP token.

---

## Runtime Resolution

When a mentor invokes an MCP server, the platform resolves credentials in this order:

```
┌────────────────────────────────────────────────────────────┐
│  1. User-scoped connection for (server, user)              │
│     ────────────────────────────────────────────────────── │
│     2. Mentor-scoped connection for (server, mentor)       │
│        ──────────────────────────────────────────────────  │
│        3. Platform-scoped connection for (server, tenant)  │
│           ───────────────────────────────────────────────  │
│           4. Featured-server fallback (global)             │
│              ────────────────────────────────────────────  │
│              5. Fail with 401 / no connection              │
└────────────────────────────────────────────────────────────┘
```

The first match wins. For OAuth connections, the linked `ConnectedService` is transparently refreshed if its access token is nearing expiry.

If `auth_scope="user"` and no user-scoped connection is found at resolution time, the platform either:

- **Triggers the in-chat OAuth prompt** (see [In-Chat MCP Events](/docs/developer/agents/mcp-authentication/in-chat-mcp-events)), or
- **Falls back to lower scopes** if `auth_scope` is `mentor` or `platform`.

---

## Front-end Notes

- **Server first, then connection.** The connection endpoints expect a valid server ID.
- **OAuth prerequisite.** Do not display a "Connect" button for `auth_type=oauth2` servers unless a `ConnectedService` exists — the API returns `400 OAuth2 connections require a connected service.` Populate the `ConnectedService` picker from `GET /api/accounts/connected-services/orgs/{org}/users/{user_id}/`.
- **Dynamic form layout.** Drive the form off the server's `auth_type`:
  - `none` — no credential fields.
  - `token` — inputs for `authorization_scheme`, `credentials`, and optional header key/value pairs.
  - `oauth2` — a `ConnectedService` picker filtered by provider and service.
- **Scope-aware fields.** Show / hide `user` and `mentor` inputs based on the selected `scope`.
- **Masked credentials.** Track whether the user changed the field locally; only send a new value when they explicitly rotate the secret.
- **Mentor wiring reminder.** After saving a connection, prompt the admin to enable `mcp-tool` and attach the server on the mentor settings screen — otherwise it will not be called in chat.

### Field-level validation errors

The API returns actionable, per-field errors — render them inline:

```json
{
  "mentor": ["Mentor scoped connections require a mentor."],
  "connected_service": ["OAuth2 connections require a connected service."]
}
```

---

## Troubleshooting

| Issue | Resolution |
| --- | --- |
| `400 Selected MCP server is not available to the current tenant.` | Bind a server scoped to the current tenant, or mark the source server `is_featured=true`. |
| `400 OAuth2 connections require a connected service.` | Complete the OAuth connector flow first; pass the resulting `connected_service` ID. |
| Connection falls back to platform credentials unexpectedly | Confirm the user connection's `is_active` flag and that `ConnectedService.user` matches the chat user. |
| Mentor ignores the configured server | Verify `mcp-tool` is in the mentor's `tools` list and the server ID is in `mcp_servers`. |
| No OAuth prompt appears for a per-user server | Set the server's `auth_scope` to `user`. `auth_type="oauth2"` alone is not enough. |
| `400 No credentials found` during connection create | Tenant admin must install `auth_{provider}` credentials before OAuth connections can be created. |
