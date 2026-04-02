# MCP & OAuth Architecture

Backend architecture reference for the MCP server and OAuth connector subsystems, covering the data model, control flow, validation, and extensibility.

---

## Overview

This document targets backend engineers who are extending or operating the MCP server and OAuth connector subsystems. It summarizes the data model, control flow, and key modules involved in provisioning external integrations.

## Component Map

| Layer | Responsibility | Key Modules |
| --- | --- | --- |
| Persistence | Store providers, services, connections, and user tokens | `OauthProvider`, `OauthService`, `ConnectedService`, `MCPServer`, `MCPServerConnection` |
| API (Accounts) | Orchestrate OAuth flows, expose discovery endpoints, manage connected services | Account views and URLs |
| API (Mentor) | CRUD for MCP servers and connections | `MCPServerViewSet`, `MCPServerConnectionViewSet` |
| LangChain Runtime | Resolve connections at inference time, build headers | `MCPServer.resolve_connection`, LangChain tools |
| Utilities | Credential store access, token refresh logic | `get_cred`, connected services utilities |

## Data Model Relationships

- `OauthProvider` exposes one or more `OauthService` entries.
- `OauthService` grants `ConnectedService` records (per user, per platform).
- `MCPServer` is registered on a `Platform` and has one or more `MCPServerConnection` entries.
- `MCPServerConnection` optionally uses a `ConnectedService` (for OAuth-backed auth) and can be scoped to a `Mentor`.
- `ConnectedService` has a unique constraint on `(user, provider, platform, service)` to enforce one connection per surface.

### Connection scope rules

- `scope="platform"` -- `platform` is auto-filled from the server's platform (unless featured).
- `scope="user"` -- Requires `user` or `connected_service`.
- `scope="mentor"` -- Requires a mentor; optionally reuses platform-level credentials when no mentor-specific ones exist.
- `auth_type="oauth2"` -- Always requires `connected_service`.

## OAuth Flow Internals

### Discovery

`OAuthServiceListAPIView` surfaces enabled services. `OAuthServiceScopeListAPIView` narrows down to a specific service scope.

### Start

The start view resolves scopes, builds the provider adapter, and stores a UUID-based hash in the cache to guard the callback. Cache entries expire after one hour to limit replay windows.

### Callback

The callback view verifies state, exchanges the code for a token, normalizes the scope/service identifier, and syncs metadata. Tokens are stored raw (`access_token`, `refresh_token`, `token_type`) and lowercased for schema consistency.

### Refresh

Downstream modules call the refresh function which uses the provider adapter to refresh tokens and re-sync metadata.

Error handling is intentionally strict: multiple services per connection raise `ValueError` early; missing tenant credentials return `404`/`400` responses.

## MCP Server Runtime Resolution

```
LangChain Tool -> MCPServer.resolve_connection(platform, user, mentor)
                   |
                   |- User-scoped connection exists?
                   |    YES -> Load connection -> Optional ConnectedService (if oauth2)
                   |
                   |- Mentor-scoped connection exists?
                   |    YES -> Use mentor credentials
                   |
                   |- Platform-scoped connection exists?
                   |    YES -> Use stored platform credentials
                   |
                   |- Server is featured?
                   |    YES -> Use global connection
                   |    NO  -> Fail with 401 / no connection
                   |
                   -> render_headers(access_token?)
                      If oauth2: ensure fresh access token (refresh if needed)
                      Return: authorization headers
```

Async code paths use `aresolve_connection()` with `.afirst()` queries to avoid blocking the event loop. Headers are merged with `extra_headers`, and explicit credentials override anything pre-existing.

## Validation and Security

### Credential sourcing

`get_cred("auth_{provider}", tenant)` is the single source of truth for client IDs and secrets. Tenant-specific overrides live alongside global (`tenant="main"`) credentials. The serializer guard ensures the admin has installed the necessary client credentials before connections are created.

### State integrity

Cache entries expire after one hour. The callback splits `state` into `(org, provider, service, user_id, hash)` and verifies the cached value before exchanging the token.

### Access control

- OAuth endpoints restrict methods (`GET`, `DELETE`) to read-only client operations, leaving creation solely to the managed OAuth flow.
- MCP endpoints require tenant admin privileges for all create/update/delete operations.

## Extensibility

### Adding a new OAuth provider/service

1. Seed `OauthProvider` and `OauthService` records via the management command.
2. Provision client credentials (`auth_{provider}`) in the credential store.
3. Update the front-end to surface the new service (the API is discovery-driven, so minimal UI changes are needed).

### Extending MCP authentication types

1. Add a new enum member to `MCPServer.AuthType`.
2. Extend connection validation to handle the new type.
3. Update `render_headers()` to format the outbound headers.
4. Update front-end forms and documentation.

### Supporting multi-tenant shared servers

Mark the server `is_featured=true`. Platform admins in other tenants can then create their own connections, while the original tenant retains control of the server metadata.
