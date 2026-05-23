# OAuth Connector API

Let learners and tenant admins grant the platform permission to act on their behalf via OAuth, powering authenticated MCP server connections.

---

## Overview

Front-end clients use the OAuth connector APIs to let a learner (or tenant admin) grant our platform permission to act on their behalf. This guide documents the discovery endpoints, the authorization handshake, and the post-connection management APIs.

**Key terms:**
- **OAuth Provider** — A top-level vendor such as Google or Dropbox.
- **OAuth Service** — A concrete surface offered by a provider (e.g., Google Drive, Google Calendar). A `ConnectedService` references one service.
- **ConnectedService** — The persisted token bundle that represents a user's grant. These are always per user *and* per service.

> **Next step:** Once a `ConnectedService` exists, wire it into an MCP server with the [MCP Server Connections](/docs/developer/agents/mcp-authentication/mcp-connections) guide, or let a learner connect inline via [In-Chat MCP Events](/docs/developer/agents/mcp-authentication/in-chat-mcp-events).

## API Summary

| Capability | Endpoint | Method | Notes |
| --- | --- | --- | --- |
| List available services | `/api/accounts/orgs/{org}/oauth-services/` | `GET` | Returns all enabled services across all providers. |
| List scopes for a service | `/api/accounts/orgs/{org}/oauth-services/{service_name}/scopes/` | `GET` | Service-specific breakdown of scopes. |
| Start OAuth flow | `/api/accounts/connected-services/orgs/{org}/users/{user_id}/{provider}/{service}/` | `GET` | Returns the vendor authorization URL and primes the OAuth state cache. |
| Handle callback | `/api/accounts/connected-services/callback/` | `GET` | Called by the browser after the vendor redirects back with a code. |
| List a user's connections | `/api/accounts/connected-services/orgs/{org}/users/{user_id}/` | `GET` | Returns `ConnectedService` records for the current user. |
| Delete a connection | `/api/accounts/connected-services/orgs/{org}/users/{user_id}/{id}/` | `DELETE` | Removes the connection. |

Authentication is the standard `Authorization: Token ...` scheme. Tenants must have credentials named `auth_{provider}` configured in the credential store before the flow can start.

## Connection Lifecycle

```
Front-end Client          Agent API (Accounts)          OAuth Provider
      |                          |                              |
      |-- GET /oauth-services/ ->|                              |
      |<- 200 OK (services) ----|                              |
      |                          |                              |
      |-- GET /connected-services/.../provider/service/ ------->|
      |<- 200 OK {"auth_url":..} |                              |
      |                          |                              |
      |-- Redirect user -------->|                              |
      |                          |<-- Redirect back w/ code ----|
      |-- GET /callback?code=... |                              |
      |                          |-- Token exchange ----------->|
      |                          |<-- Token payload ------------|
      |<- 200 OK (Connected) ---|                              |
```

## Step-by-Step Implementation

### Discover services

```
GET /api/accounts/orgs/acme/oauth-services/ HTTP/1.1
Authorization: Token {{TOKEN}}
```

Sample response:

```json
[
  {
    "id": 12,
    "oauth_provider": "google",
    "name": "drive",
    "display_name": "Google Drive",
    "description": "File access for Drive",
    "scope": "https://www.googleapis.com/auth/drive",
    "image": "https://cdn.example.com/oauth/google-drive.svg",
    "created_at": "2025-11-01T12:32:55Z",
    "updated_at": "2025-11-01T12:32:55Z"
  }
]
```

### Start the OAuth flow

When the learner clicks **Connect**, call the start endpoint:

```
GET /api/accounts/connected-services/orgs/acme/users/alice/google/drive/ HTTP/1.1
Authorization: Token {{TOKEN}}
```

Response:

```json
{
  "auth_url": "https://accounts.google.com/o/oauth2/v2/auth?client_id=..."
}
```

Open `auth_url` in a new window/tab. The user authenticates with the vendor and approves permissions.

### Handle the callback

After approval, the vendor redirects the browser to the callback URL. Capture the query parameters and relay them:

```
GET /api/accounts/connected-services/callback/?code=4/0A...&state=acme:google:drive:alice:09e4... HTTP/1.1
```

Successful response:

```json
{
  "id": 77,
  "provider": "google",
  "service": "drive",
  "expires_at": "2025-11-12T14:05:00Z",
  "scope": "https://www.googleapis.com/auth/drive",
  "scope_names": ["drive"],
  "scopes": ["https://www.googleapis.com/auth/drive"],
  "token_type": "bearer",
  "service_info": {
    "id": 12,
    "name": "drive",
    "display_name": "Google Drive",
    "logo": "https://cdn.example.com/oauth/google-drive.svg"
  }
}
```

At this point the connection is persisted. If the connection already existed, it is updated in place.

### Listing and deleting connections

```
GET /api/accounts/connected-services/orgs/acme/users/alice/ HTTP/1.1
Authorization: Token {{TOKEN}}
```

To delete:

```
DELETE /api/accounts/connected-services/orgs/acme/users/alice/77/ HTTP/1.1
Authorization: Token {{TOKEN}}
```

Returns `204 No Content`.

## UI/UX Considerations

- **State management** -- The `state` returned by the start endpoint must be round-tripped without modification. Do not decode or alter it.
- **Window strategy** -- Use `window.open` or a redirect. If using a modal, capture the redirect in that context and forward the query params to the callback endpoint.
- **Error handling** -- Handle HTTP 400 responses (most likely missing provider credential configuration). Display actionable guidance ("Admin must configure auth_google credentials").
- **Credential refresh** -- The API auto-refreshes tokens when needed. The front-end only needs to re-list connections to pick up new expiry times.

## Troubleshooting

| Symptom | Likely Cause | Action |
| --- | --- | --- |
| `/oauth-services/` returns empty array | Tenant has no enabled services or provider is disabled | Confirm `OauthProvider.is_enabled` and `OauthService` records. |
| Start endpoint returns 400 `"No credentials found"` | `auth_{provider}` credential is missing | Ask tenant admin to configure credential via admin UI. |
| Callback returns `Invalid state` | Start and callback requests happened in different browser contexts or state expired | Ensure the same browser session completes the round-trip within 60 minutes. |
| Callback returns `Could not exchange auth token` | Provider rejected the code | Try restarting the flow; verify redirect URI matches provider settings. |
