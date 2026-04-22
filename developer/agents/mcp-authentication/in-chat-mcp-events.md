# In-Chat MCP Events

Handle the real-time WebSocket and SSE events the mentor runtime emits when an MCP server needs attention during a live chat — per-user OAuth prompts, tool-retrieval recoveries, and graceful failures.

---

## Overview

Most MCP authentication happens well before a learner opens a chat: an administrator provisions a server and a connection, and every request uses those credentials. Some integrations, however, need to authenticate **each learner individually**. For these, the platform pauses the active chat and asks the user to authenticate in a side window, then resumes the conversation automatically once credentials arrive.

This doc describes the messages the backend emits on the existing chat connection during that handshake (and other MCP lifecycle events), so that your client can react correctly.

The in-chat OAuth flow only triggers when **all** of the following are true:

- The MCP server has `auth_type = "oauth2"`.
- The MCP server has `auth_scope = "user"`.
- No valid `MCPServerConnection` exists for the current user + server.
- The chat session is an authenticated (non-anonymous) user.

For a full walkthrough of server configuration and connection provisioning, see [MCP Server Connections](/docs/developer/agents/mcp-authentication/mcp-connections).

---

## Prerequisites

### Server Configuration

| Requirement | Detail |
| --- | --- |
| **MCP Server** | Registered with `is_enabled=true` and attached to the mentor |
| **`auth_type`** | Must be `"oauth2"` |
| **`auth_scope`** | Must be `"user"` for the in-chat flow |
| **`oauth_service`** | Must link to a valid `OauthService` → `OauthProvider` |
| **Client credentials** | `auth_{provider}` credentials must exist in the credential store with `client_id`, `client_secret`, and a `redirect_uri` |
| **Active session** | The chat session must belong to an authenticated user |

### Auth scope matrix

The `auth_scope` field on the MCP server controls whether the in-chat prompt fires at all.

| Value | Behavior |
| --- | --- |
| `platform` (default) | Uses shared, pre-provisioned platform credentials. No prompt. |
| `mentor` | Uses mentor-scoped credentials. No prompt. |
| `user` | **Each user authenticates individually.** The in-chat flow runs when no user-scoped connection exists. |

---

## Sequence Diagram

```
Frontend (WS / SSE)         Backend                       OAuth Provider
      │                        │                                 │
      │── user sends message ─▶│                                 │
      │                        │── resolve server headers ─┐     │
      │                        │   (no user connection)    │     │
      │                        │◀──────────────────────────┘     │
      │                        │                                 │
      │◀── oauth_required ─────│                                 │
      │   {auth_url, server_*} │                                 │
      │                        │── begin polling (every 10s) ──▶ │
      │                        │                                 │
      │── open auth_url ──────▶│────── redirect to provider ───▶ │
      │                        │                                 │
      │                        │     [user consents]             │
      │                        │                                 │
      │                        │◀──── callback with code ────────│
      │                        │───── exchange for tokens ─────▶ │
      │                        │◀──── access / refresh tokens ───│
      │                        │                                 │
      │                        │── create ConnectedService ─┐    │
      │                        │   + MCPServerConnection    │    │
      │                        │◀───────────────────────────┘    │
      │                        │                                 │
      │◀── oauth_connection_*──│                                 │
      │   resolved             │                                 │
      │                        │─── resume chat processing ───▶  │
      │◀── normal chat reply ──│                                 │
```

If the learner does not finish within the timeout window:

```
      │                        │── polling exhausted (5 min) ──▶ │
      │◀── error (400) ────────│                                 │
      │   "Timed out waiting…" │                                 │
```

---

## Event Reference

Every event arrives as a JSON string on the existing chat WebSocket/SSE connection. Parse it with `JSON.parse()` and switch on the `type` field.

### `oauth_required`

**Sent when:** The backend is about to call an MCP server for which the current user has no connection. Arrives **before** polling starts.

**Client action:** Show the user the `auth_url` (link or popup) so they can complete authentication with the external provider.

```json
{
  "type": "oauth_required",
  "server_name": "Google Drive MCP",
  "server_id": 42,
  "auth_url": "https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=...&response_type=code&scope=...&state=...",
  "message": "Authentication required for MCP server 'Google Drive MCP'. Please complete the OAuth flow to continue."
}
```

| Field | Type | Description |
| --- | --- | --- |
| `type` | `string` | Always `"oauth_required"`. |
| `server_name` | `string` | Human-readable name of the MCP server. |
| `server_id` | `integer` | Database ID of the MCP server. |
| `auth_url` | `string` | The full OAuth authorization URL. |
| `message` | `string` | User-safe explanation. |

**Client guidance**

- Render a prominent UI element (banner, modal, or inline card) naming the server.
- Open `auth_url` in a new tab or popup — OAuth providers typically block framing.
- Show a "waiting for authentication" indicator while the backend polls.
- Do **not** close or refresh the chat connection; the resolution event will arrive on the same socket.

---

### `oauth_connection_resolved`

**Sent when:** The backend's polling loop detects that the OAuth flow succeeded and a valid `MCPServerConnection` now exists for the user.

**Client action:** Dismiss the OAuth prompt UI. The chat will resume automatically.

```json
{
  "type": "oauth_connection_resolved",
  "server_name": "Google Drive MCP",
  "server_id": 42,
  "message": "OAuth connection resolved for MCP server 'Google Drive MCP'. Continuing with chat."
}
```

| Field | Type | Description |
| --- | --- | --- |
| `type` | `string` | Always `"oauth_connection_resolved"`. |
| `server_name` | `string` | Name of the authenticated server. |
| `server_id` | `integer` | Database ID of the authenticated server. |
| `message` | `string` | Friendly confirmation message. |

**Client guidance**

- Hide the OAuth prompt.
- Optionally show a success toast (e.g., "Connected to Google Drive MCP").
- The chat response arrives shortly after this event.

---

### `mcp_tools_retrieved`

**Sent when:** An initial MCP tool fetch failed but a retry succeeded. The backend retries up to 3 times with exponential backoff (`1s`, `2s`, `4s`).

**Client action:** None required. This is purely informational.

```json
{
  "type": "mcp_tools_retrieved",
  "session_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "mentor_id": "123"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `type` | `string` | Always `"mcp_tools_retrieved"`. |
| `session_id` | `string` | Current chat session UUID. |
| `mentor_id` | `string` | Mentor ID for the session. |

**Client guidance**

- Log for debugging or ignore.
- Optionally surface a subtle indicator that tools recovered.

---

### `warning`

**Sent when:** The backend fails to load MCP tools for a reason **unrelated** to OAuth — the server is unreachable, configuration is invalid, or retries are exhausted. The chat continues without MCP tools (graceful degradation).

**Client action:** Inform the user that some tools may be unavailable, but keep the chat going.

```json
{
  "type": "warning",
  "message": "MCP tools temporarily unavailable for this session. Continuing without them.",
  "developer_error": "ConnectionError: MCP server unreachable",
  "code": 503
}
```

| Field | Type | Description |
| --- | --- | --- |
| `type` | `string` | Always `"warning"`. |
| `message` | `string` | User-safe message for display. |
| `developer_error` | `string` | Technical detail for logging. **Do not show to end users.** |
| `code` | `integer` | HTTP-equivalent status (`503` = Service Unavailable). |

**Client guidance**

- Show a non-blocking banner or toast using `message`.
- Log `developer_error` for diagnostics.
- The chat reply still arrives, but without MCP-powered capabilities.
- The user can retry the message once the external service recovers.

---

### `error` (ChatValidationError)

**Sent when:** A validation failure terminates the chat turn. In the MCP context, this happens in three situations:

1. **OAuth timeout** — the user did not finish authentication within `MCP_OAUTH_MAX_WAIT_SECONDS` (default 5 minutes).
2. **OAuth URL build failure** — the backend could not construct the authorization URL (missing credentials, unknown provider, etc.).
3. **Missing connected service** — an OAuth2 connection record exists but has no linked `ConnectedService` with valid tokens.

**Client action:** Show the error to the user and offer a retry.

```json
{
  "error": "Timed out waiting for OAuth authentication for MCP server 'Google Drive MCP' after 300s. Retry message after completing the OAuth flow.",
  "status_code": 400
}
```

| Field | Type | Description |
| --- | --- | --- |
| `error` | `string` | User-facing error description. |
| `status_code` | `integer` | Always `400` for `ChatValidationError`. |

**Example payloads**

*OAuth URL build failure*

```json
{
  "error": "Could not build OAuth URL for MCP server 'Google Drive MCP'.",
  "status_code": 400
}
```

*Missing connected service*

```json
{
  "error": "MCP connection for server 'Google Drive MCP' is configured for OAuth2 but has no connected service.",
  "status_code": 400
}
```

**Client guidance**

- Display `error` directly; it is written for end users.
- For timeouts, the prior `oauth_required` prompt still contained the `auth_url`. If the user finishes OAuth *after* the timeout, their next chat message will succeed automatically because the connection is now in place.
- Offer a **Retry** button so the user can resend their message.
- On WebSocket transports, the connection closes after this error.

---

## Error Handling Summary

| Scenario | Event | `status_code` | Chat continues? |
| --- | --- | --- | --- |
| OAuth prompt sent | `oauth_required` | — | Paused (backend polls) |
| User completes OAuth | `oauth_connection_resolved` | — | Yes (resumes) |
| Tools retrieved after retry | `mcp_tools_retrieved` | — | Yes |
| Tools unavailable (non-OAuth) | `warning` | `503` | Yes, without MCP tools |
| OAuth timeout (5 min) | `error` | `400` | No — user must retry |
| OAuth URL build failure | `error` | `400` | No — user must retry |
| Missing connected service | `error` | `400` | No — user must retry |

---

## Timing & Polling Constants

| Constant | Value | Description |
| --- | --- | --- |
| `MCP_OAUTH_MAX_WAIT_SECONDS` | `300` (5 min) | Maximum time the backend waits for the user to finish OAuth. |
| `MCP_OAUTH_POLL_INTERVAL_SECONDS` | `10` | Interval between DB polls for new credentials. |
| MCP retry attempts | `3` | Number of retries for transient MCP tool retrieval failures. |
| MCP retry backoff | `1s`, `2s`, `4s` | Exponential backoff between retries. |

Polling starts immediately after `oauth_required` is sent. Each poll checks for:

1. An `MCPServerConnection` with a valid `ConnectedService` for the user + server, or
2. A `ConnectedService` matching the OAuth provider + user + platform.

On the first match, the connection resolves and `oauth_connection_resolved` is emitted. On timeout, a `ChatValidationError` is raised.

---

## Frontend Implementation Guide

### Message handler pattern

```javascript
// Inside your WebSocket/SSE message handler
function handleMessage(data) {
  const message = JSON.parse(data);

  switch (message.type) {
    case "oauth_required":
      showOAuthPrompt({
        serverName: message.server_name,
        serverId: message.server_id,
        authUrl: message.auth_url,
        displayMessage: message.message,
      });
      break;

    case "oauth_connection_resolved":
      dismissOAuthPrompt(message.server_id);
      showToast(`Connected to ${message.server_name}`);
      break;

    case "mcp_tools_retrieved":
      console.debug("MCP tools recovered", message);
      break;

    case "warning":
      showWarningBanner(message.message);
      console.warn("MCP warning:", message.developer_error);
      break;

    default:
      if (message.error && message.status_code) {
        handleError(message);
      }
      break;
  }
}
```

### OAuth prompt UX tips

- Show the **MCP server name** prominently so the user knows what they are granting access to.
- Open `auth_url` in a **new tab or popup**, not an iframe — OAuth providers block framing.
- Show a waiting state after the user clicks the link so the chat doesn't appear stuck.
- **Auto-dismiss** on `oauth_connection_resolved`.
- **Show the timeout message** if an `error` arrives while the prompt is visible, and offer a retry.
- Let the user **manually dismiss** the prompt and retry their message on demand.

### Handling the post-OAuth redirect

The OAuth callback URL redirects the browser back to the application; the backend handles the token exchange automatically. The frontend does **not** need to process the callback itself — it only needs to listen for `oauth_connection_resolved` on the open chat connection.

If the callback opens in a popup, you may auto-close it after the redirect completes. The main chat tab receives `oauth_connection_resolved` regardless of whether the popup is still open.

---

## Troubleshooting

| Symptom | Likely Cause | Action |
| --- | --- | --- |
| No `oauth_required` event ever arrives | Server is not configured with `auth_type="oauth2"` + `auth_scope="user"` | Update the server via `PATCH /mcp-servers/{id}/` with both fields. |
| `oauth_required` fires every message even after the user authenticates | Resolved connection belongs to a different user or platform | Check that the callback created a `ConnectedService` for the chat user and the correct tenant. |
| Timeout with no UI shown | Client didn't handle `oauth_required` — event was ignored | Ensure the chat message handler dispatches on `message.type`, not just content. |
| `error` with "Could not build OAuth URL" | Missing `auth_{provider}` credential in the credential store | Configure the provider credential (client ID, secret, redirect URI) as a tenant admin. |
| Chat continues but tool call fails silently | `warning` with `503` was emitted and ignored | Surface the warning in the UI, and verify the MCP server is reachable from the platform. |

---

## Related Documentation

- [MCP Server Connections](/docs/developer/agents/mcp-authentication/mcp-connections) — register servers, create connections, and attach them to mentors.
- [OAuth Connectors](/docs/developer/agents/mcp-authentication/oauth-connectors) — provider/service discovery and the standalone OAuth handshake used outside of chat.
- [Architecture](/docs/developer/agents/mcp-authentication/architecture) — backend internals, data model, and the runtime resolution pipeline.
