# ibl.ai Platform Integration — Deep Dive (Component 4)

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/platform-integration.md`](https://github.com/iblai/ontology/blob/main/docs/platform-integration.md). This page is generated — edit it in the repository, not here.

> Part of the iblai-ontology architecture. See the architecture overview, the [outbound exposure component](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md), and the [identity deep dive](/developer/guides/ontology-identity).

The on-premise knowledge layer exposes itself as an MCP server ([Component 4](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md)). On the ibl.ai side, the platform's **existing** MCP Server Connections infrastructure handles registration, authentication, credential resolution, and token refresh automatically. iblai-ontology requires **no custom authentication infrastructure** — we point the platform's OAuth machinery at Entra ID instead of Google or Dropbox.

Everything below reflects the live ibl.ai codebase: it models MCP Server / MCP Server Connection / Connected Service, implements `discover_and_register_mcp_oauth_service` using **RFC 9728** (Protected Resource Metadata discovery) and **RFC 7591** (Dynamic Client Registration), and forwards the user's role via `extra_headers` as `X-Iblai-Role`.

---

## The Data Model

The platform represents the integration with three objects:

| Object | What it is |
|---|---|
| **MCP Server** | Metadata describing the endpoint: name, URL, transport, `auth_type`, `auth_scope`. |
| **MCP Server Connection** | An authentication binding between a tenant / agent / user and the server. |
| **Connected Service** | The persisted OAuth token bundle for per-user connections (access + refresh tokens, expiry). |

Two fields on the MCP Server answer different questions:

| Field | Answers | Values |
|---|---|---|
| `auth_type` | How is the call authenticated? | `none` / `token` / `oauth2` |
| `auth_scope` | Whose credentials are used? | `platform` / `agent` / `user` |

Registration endpoints live under:

```
base.manager.iblai.app/api/ai-agent/orgs/<org>/users/admin/mcp-servers/
base.manager.iblai.app/api/ai-agent/orgs/<org>/users/admin/mcp-server-connections/
```

---

## Registering iblai-ontology

### Step 1 — Register the MCP Server

```bash
curl -X POST "https://base.manager.iblai.app/api/ai-agent/orgs/alasu/users/admin/mcp-servers/" \
  -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iblai-ontology",
    "description": "On-premise knowledge layer: student records, enrollment, financial aid, courses, advising, facilities",
    "url": "https://ontology.alasu.edu/mcp",
    "transport": "streamable_http",
    "auth_type": "oauth2",
    "auth_scope": "user",
    "is_enabled": true
  }'
```

Response (note the `id` — used everywhere below):

```json
{
  "id": 14, "platform": 87, "name": "iblai-ontology",
  "url": "https://ontology.alasu.edu/mcp",
  "transport": "streamable_http",
  "auth_type": "oauth2", "auth_scope": "user",
  "is_featured": false, "is_enabled": true,
  "created_at": "2026-06-20T12:00:00Z"
}
```

The key choices:

- `auth_type: "oauth2"` — the gateway expects Entra ID OAuth tokens, not static API keys.
- `auth_scope: "user"` — **each user authenticates individually** with their university Entra ID account. This is essential because permissions are per-user (a counselor sees different data than a student).
- `transport: "streamable_http"` — MCP over HTTPS, which works through the university firewall.

### Step 2 — Configure the OAuth provider (Entra ID)

The platform needs the university's Entra tenant credentials in the same credential store it uses for any OAuth provider:

```
Key:    auth_microsoft
Tenant: alasu
Value:  {
  "client_id":     "<iblai-ontology-app-client-id>",
  "client_secret": "<ontology-app-client-secret>",
  "redirect_uri":  "https://base.manager.iblai.app/api/accounts/connected-services/callback/",
  "tenant_id":     "<alasu-entra-tenant-id>"
}
```

An `OauthProvider` (`"microsoft"`) and `OauthService` (`"ontology"`) are registered with the right scopes:

```json
{
  "oauth_provider": "microsoft",
  "name": "ontology",
  "display_name": "University Data Access",
  "scope": "api://<iblai-ontology-app-client-id>/IblaiOntology.Read openid profile email"
}
```

This is where RFC 9728 / RFC 7591 come in: `discover_and_register_mcp_oauth_service` can discover the protected-resource metadata and dynamically register a client, so the OAuth wiring is largely automatic.

### Step 3 — Attach the server to agents

```bash
curl -X PATCH "https://base.manager.iblai.app/api/ai-agent/orgs/alasu/users/admin/agents/$FINAID_AGENT_ID/settings/" \
  -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "tools": ["mcp-tool"], "mcp_servers": [14] }'
```

Repeat for the Advising Agent, Registrar Agent, Student Self-Service Agent, etc. Once the MCP tool is enabled and the server attached, the agent discovers iblai-ontology's tools and calls them in its responses.

---

## Assigning Roles via Connections

Role assignment is done by creating MCP Server Connections with the role in `extra_headers`. The gateway reads that header as `X-Iblai-Role` (see [identity.md](/developer/guides/ontology-identity)).

```bash
# User-scoped: this user gets this role
curl -X POST ".../mcp-server-connections/" -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" -d '{
    "server": 14, "scope": "user", "auth_type": "oauth2",
    "user": "tsmith", "connected_service": 77,
    "extra_headers": { "X-Iblai-Role": "FinancialAidCounselor" }
  }'

# Agent-scoped default role (used when no user-specific connection exists)
curl -X POST ".../mcp-server-connections/" -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" -d '{
    "server": 14, "scope": "agent", "auth_type": "oauth2",
    "agent": "<finaid-agent-uuid>", "connected_service": 80,
    "extra_headers": { "X-Iblai-Role": "FinancialAidCounselor" }
  }'
```

Bulk onboarding (a department at a time) is a CSV import in the platform admin UI mapping email → role:

```
email,role
tsmith@alasu.edu,FinancialAidCounselor
rsmith@alasu.edu,AcademicAdvisor
jackie@alasu.edu,Registrar
sonia@alasu.edu,Executive
```

---

## Per-User Authentication: The In-Chat OAuth Flow

Because `auth_scope = "user"` and `auth_type = "oauth2"`, the platform automatically triggers an in-chat OAuth flow the *first* time a user interacts with an agent that needs iblai-ontology:

```
1. Counselor opens chat with the Financial Aid Agent
2. Asks: "What is Jane Doe's aid package status?"
3. Agent needs ontology data -> MCPServer.resolve_connection(platform, user, agent)
   -> no MCPServerConnection exists yet for this user + server
4. Platform emits an oauth_required event to the chat UI with an Entra ID auth_url
5. Chat UI shows a "Connect University Account" button
6. Counselor clicks -> Entra ID login (email + password + MFA) -> authorization code
7. Browser redirects to the ibl.ai callback. The platform:
     a. verifies state
     b. exchanges the code for tokens with Entra ID
     c. creates a ConnectedService { provider, service, user, access_token, refresh_token, expires_at }
     d. creates an MCPServerConnection { server:14, scope:"user", user, connected_service }
8. Platform's polling loop (every ~10s) detects the new connection
   -> emits oauth_connection_resolved
9. Chat resumes automatically; the platform renders:
     Authorization: Bearer <counselor's Entra ID access token>
     X-Iblai-Role: FinancialAidCounselor
10. On-prem gateway validates the JWT, resolves the role via roles.yaml, runs the tool, returns scoped data
11. Agent answers the question
```

Subsequent interactions skip steps 4–8. The platform refreshes the Entra access token via the stored refresh token before it expires — the user never sees an expiration prompt. If the user does not finish authenticating within the window (`MCP_OAUTH_MAX_WAIT_SECONDS`, default ~5 min), the platform emits an error event and the user can retry.

---

## Runtime Resolution Chain

When any agent invokes the iblai-ontology MCP server, the platform resolves credentials in priority order — **first match wins**:

```
Agent calls MCP tool
  -> MCPServer.resolve_connection(platform, user, agent)
     |
     |- 1. User-scoped connection (server=14, user=current_user)?
     |       YES -> load ConnectedService -> refresh Entra token if near expiry
     |              -> render_headers(access_token) -> forward to https://ontology.alasu.edu/mcp
     |
     |- 2. Agent-scoped connection (server=14, agent=current_agent)?
     |       YES -> use agent-specific credentials
     |
     |- 3. Platform-scoped connection (server=14, platform=alasu)?
     |       YES -> use shared platform credentials
     |
     |- 4. No connection, auth_scope="user"?
     |       YES -> trigger in-chat OAuth (oauth_required event)
     |
     |- 5. No connection, auth_scope != "user"?
             -> fail with 401 / no connection
```

For OAuth connections, the linked `ConnectedService` is transparently refreshed when its access token nears expiry.

---

## Platform-Scoped (Service Account) Connections

For agents needing broad, non-user-specific access (e.g. a nightly analytics agent), register a *second* MCP server entry with a static/client-credentials token:

```bash
# Service-account server entry
curl -X POST ".../mcp-servers/" -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" -d '{
    "name": "iblai-ontology (Service)", "url": "https://ontology.alasu.edu/mcp",
    "transport": "streamable_http", "auth_type": "token", "auth_scope": "platform",
    "is_enabled": true }'
# -> { "id": 15, ... }

# Platform-scoped connection with a service-account token
curl -X POST ".../mcp-server-connections/" -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" -d '{
    "server": 15, "scope": "platform", "auth_type": "token",
    "credentials": "<entra-id-client-credentials-token>",
    "authorization_scheme": "Bearer" }'
```

The gateway recognizes client-credentials tokens (Entra application permissions) and maps them to a service role with appropriate scope limits.

---

## Agent-Scoped Connections

Different agents can have different access to the same ontology, via agent-scoped connections:

```bash
curl -X POST ".../mcp-server-connections/" -H "Authorization: Token $ADMIN_TOKEN" \
  -H "Content-Type: application/json" -d '{
    "server": 15, "scope": "agent", "auth_type": "token",
    "agent": "<finance-agent-uuid>",
    "credentials": "<finance-service-account-token>",
    "authorization_scheme": "Bearer" }'
```

---

## Why This Integration Works

- **No custom authentication infrastructure.** The platform already has provider registration, token exchange, callback handling, refresh, and credential resolution. We just point it at Entra ID.
- **Per-user identity flows automatically.** In-chat OAuth prompts each user exactly once; after that, every MCP call carries their identity.
- **The gateway never needs to know about the platform.** It only validates Entra ID tokens — whether they came from the platform, a university app, or a direct client is irrelevant. (This is what makes [Option B](/developer/guides/ontology-identity#option-b--per-user-oauth-directly-on-iblai-ontology-notion-model) possible without changing the gateway.)
- **Fallbacks at every scope.** Background agents use service accounts (platform-scoped); department agents use agent-scoped credentials; interactive agents use per-user credentials.
- **Audit spans both systems.** The platform logs who triggered which call; the gateway logs token validation and data access with the JWT's `jti` for end-to-end traceability.
- **Token refresh is invisible.** The platform refreshes Entra tokens automatically.

---

## Related

- Outbound exposure (the on-prem side): [components/04-mcp-outbound.md](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md)
- Token validation, `roles.yaml`, Option A vs B: [identity.md](/developer/guides/ontology-identity)
- Registering the server end-to-end as part of rollout: [deployment.md](/developer/guides/ontology-deployment)
