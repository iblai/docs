# MCP Server Connections

Register MCP servers, wire them to a mentor, and create authentication bindings so AI mentors can securely invoke external tools at runtime.

---

## Overview

The Model Context Protocol (MCP) lets an AI mentor call external tools — anything from a Google Drive search to a custom workflow engine — as part of a conversation. The ibl.ai platform models MCP in three objects:

| Term | Description |
| --- | --- |
| **MCP Server** | Metadata describing an external MCP endpoint (name, URL, transport, auth type, auth scope). Tenants can register any number of servers. |
| **MCP Server Connection** | An authentication binding between a tenant, mentor, or user and an MCP server. Stores the token or the reference to an OAuth-connected service. |
| **Connection Scope** | Whether the connection is shared tenant-wide (`platform`), bound to a specific mentor (`mentor`), or owned by a single user (`user`). |

A working MCP integration therefore requires three things:

1. A **registered server** record.
2. At least one **connection** providing credentials at the desired scope.
3. The target **mentor** must have the MCP tool enabled and the server assigned.

> **Prerequisite:** OAuth-backed MCP servers depend on the connector flow documented in the [OAuth Connectors](/docs/developer/agents/mcp-authentication/oauth-connectors) guide. Per-user OAuth servers can also authenticate learners in-chat — see [In-Chat MCP Events](/docs/developer/agents/mcp-authentication/in-chat-events).

---

## End-to-End Flow

```
                  ┌─────────────────────────────────┐
                  │  1. Register MCP Server         │
                  │     POST /mcp-servers/          │
                  └──────────────┬──────────────────┘
                                 │
                                 ▼
                  ┌─────────────────────────────────┐
                  │  2. Provide credentials         │
                  │     a. Token  → credentials blob │
                  │     b. OAuth  → OAuth Connector  │
                  └──────────────┬──────────────────┘
                                 │
                                 ▼
                  ┌─────────────────────────────────┐
                  │  3. Create MCP connection       │
                  │     scope: platform/mentor/user │
                  └──────────────┬──────────────────┘
                                 │
                                 ▼
                  ┌─────────────────────────────────┐
                  │  4. Enable `mcp-tool` on mentor │
                  │     + attach server IDs         │
                  └──────────────┬──────────────────┘
                                 │
                                 ▼
                  ┌─────────────────────────────────┐
                  │  5. Learner chats → MCP tools   │
                  │     invoked with fresh headers  │
                  └─────────────────────────────────┘
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
| `auth_scope` | `platform` (default), `mentor`, `user` | **Where the platform should look for credentials.** See below. |
| `is_featured` | `true` / `false` | If `true`, other tenants can create their own connections to this server. |
| `is_enabled` | `true` / `false` | Hard off-switch — disabled servers are skipped at runtime. |

### `auth_type` vs `auth_scope`

These two fields are often confused. They answer different questions:

| Field | Question it answers |
| --- | --- |
| `auth_type` | *How* is the call authenticated? (no auth, static token, or OAuth2) |
| `auth_scope` | *Whose credentials* should be used? (shared platform, mentor-specific, or each user's own) |

- `auth_scope="platform"` (default) — one set of credentials is shared by every chat on the tenant.
- `auth_scope="mentor"` — a specific mentor supplies its own credentials.
- `auth_scope="user"` — **each learner authenticates for themselves.** Combined with `auth_type="oauth2"`, this triggers the [in-chat OAuth flow](/docs/developer/agents/mcp-authentication/in-chat-events): when a learner first uses the mentor, they are prompted mid-chat to connect their account.

### Token-only servers

If `auth_type` is `token`, set the `credentials` field to the full header value (e.g., `"Bearer abc123"`) either at creation time or later via `PATCH /mcp-servers/{id}/`. Token-only servers normally use `auth_scope="platform"`.

---

## 2. Create Connections

A connection binds credentials to a **scope** — the realm the credentials apply in. The scope of the connection is independent of the server's `auth_scope`, but the two usually line up.

### Token-based platform connection

The simplest case — a single API key shared by everyone on the tenant.

```json
{
  "server": 9,
  "scope": "platform",
  "auth_type": "token",
  "credentials": "Token super-secret",
  "authorization_scheme": "Token",
  "extra_headers": {
    "x-mcp-client": "mentor-ui"
  }
}
```

- `authorization_scheme` becomes the prefix in the `Authorization` header.
- `extra_headers` is an arbitrary JSON object merged into every request.

### OAuth-based user connection

1. Ensure a [`ConnectedService`](/docs/developer/agents/mcp-authentication/oauth-connectors) already exists for the target user, provider, and service.
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

The platform will automatically refresh the OAuth tokens as they approach expiry — no action needed from the client.

### Mentor-scoped connection

Different mentors on the same tenant can present different credentials to the same MCP server (e.g., a Finance Mentor using a read/write token while a General Mentor uses read-only).

```json
{
  "server": 9,
  "scope": "mentor",
  "auth_type": "token",
  "mentor": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "credentials": "Token scoped-to-mentor",
  "authorization_scheme": "Token"
}
```

- `mentor` is the mentor's `unique_id` (UUID).
- The mentor's `platform_key` must match the current tenant.
- `platform` is inferred from the mentor when omitted.

### Updating and deactivating connections

- `PATCH /mcp-server-connections/{id}/` — toggle `is_active`, swap the linked `connected_service`, rotate a token, or update headers.
- `DELETE /mcp-server-connections/{id}/` — removes the record entirely.

> **Masking.** For security, credential strings are masked on read (e.g., `"sk-****key"`). Only send a new value during updates if the secret was intentionally rotated.

---

## 3. Enable the MCP Tool on a Mentor

Registering a server and its connection is only half the story. The mentor must also:

1. Have the MCP tool enabled.
2. Know which server IDs it is allowed to call.

Both are set through the mentor settings endpoint.

### Enable the tool

```http
PUT /api/ai-mentor/orgs/acme/users/alice/mentors/{mentor_id}/settings/ HTTP/1.1
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
> - Passing `[]` clears all enabled tools / attached servers.
> - Passing `null` leaves the existing value untouched.

Once the tool is enabled and at least one server is attached, the mentor will discover MCP tools and call them as part of its responses.

---

## Runtime Resolution

When a mentor invokes the MCP server, the platform resolves credentials in this order:

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

If `auth_scope="user"` and no user-scoped connection exists at resolution time, the platform either:

- **Triggers the in-chat OAuth prompt** (see [In-Chat MCP Events](/docs/developer/agents/mcp-authentication/in-chat-events)), or
- **Falls back to lower scopes** if `auth_scope` is `mentor` or `platform`.

---

## Front-end Notes

- **Server first, then connection.** The connection endpoints expect a valid server ID.
- **OAuth prerequisite.** Do not display a "Connect" button for `auth_type=oauth2` servers unless the user has an existing `ConnectedService`; otherwise the API returns `400 OAuth2 connections require a connected service.`
- **Scope hints.** Drive form layout off the server's `auth_type`:
  - `none` — informational text only.
  - `token` — inputs for `authorization_scheme`, `credentials`, and optional header key/value pairs.
  - `oauth2` — a `ConnectedService` picker filtered by provider and service.
- **Masked credentials.** Never echo the masked value back on update. Track whether the user changed the field locally; only send a new value when they explicitly rotate the secret.
- **Mentor wiring.** After saving a connection, remind admins to enable `mcp-tool` and attach the server on the mentor settings screen — otherwise it will not be called in chat.

---

## Troubleshooting

| Issue | Resolution |
| --- | --- |
| `400 Selected MCP server is not available to the current tenant.` | Bind a server scoped to the current tenant, or mark the source server `is_featured=true`. |
| `400 OAuth2 connections require a connected service.` | Complete the OAuth connector flow first; pass the resulting `connected_service` ID. |
| Connection falls back to platform credentials unexpectedly | Confirm the user connection's `is_active` flag and that `ConnectedService.user` matches the chat user. |
| Mentor ignores the configured server | Verify `mcp-tool` is in the mentor's `tools` list and the server ID is in `mcp_servers`. |
| No OAuth prompt appears for a per-user server | Set the server's `auth_scope` to `user`. `auth_type` alone is not enough. |
