# Identity & Permissions — Deep Dive (Component 3)

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/identity.md`](https://github.com/iblai/ontology/blob/main/docs/identity.md). This page is generated — edit it in the repository, not here.

> Part of the iblai-ontology architecture. See the architecture overview and the [component summary](https://github.com/iblai/ontology/blob/main/docs/components/03-identity.md).

Identity is the most critical component. The SSO system must authenticate *every* request — from the ibl.ai agent runtime, a university-built application, or a direct API call — and carry the user's identity and permissions all the way down to individual MCP tool invocations.

This repo implements **Option A first** (the gateway trusts an `X-Iblai-Role` header set by the ibl.ai platform). **Option B** (on-prem OAuth, Notion-style) is documented as the roadmap alternative and can coexist. Both are below.

---

## How Microsoft Entra ID Fits

Most universities run **Microsoft Entra ID** (formerly Azure AD) as their identity provider. Students, faculty, and staff authenticate with their university credentials; Entra ID issues OAuth 2.0 / OIDC tokens that encode identity (and, optionally, group memberships and custom roles).

The key simplification: **iblai-ontology only needs Entra ID for *authentication* (proving identity), not *authorization* (role assignment).** That means a basic app registration — no App Roles, no group-to-role mapping in the Azure portal.

---

## End-to-End Token Flow (Option A)

### Step 1 — User authenticates with the agent runtime

The user opens a chat interface (web app, Teams bot, mobile) backed by the ibl.ai agent runtime, which redirects to Entra ID:

```
https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/authorize?
  client_id=<iblai-ontology-app-client-id>
  &response_type=code
  &redirect_uri=https://agents.ibl.ai/auth/callback
  &scope=openid profile email api://<iblai-ontology-app-client-id>/IblaiOntology.Read
  &state=<random-state>
```

The user enters their university email + password (+ MFA). Entra ID issues an authorization code.

### Step 2 — Agent runtime exchanges the code for tokens

```bash
curl -X POST https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token \
  -d "client_id=<iblai-ontology-app-client-id>" \
  -d "client_secret=<ontology-app-secret>" \
  -d "grant_type=authorization_code" \
  -d "code=<authorization-code>" \
  -d "redirect_uri=https://agents.ibl.ai/auth/callback" \
  -d "scope=openid profile email api://<iblai-ontology-app-client-id>/IblaiOntology.Read"
```

The access token (JWT) carries identity claims:

```json
{
  "aud": "api://<iblai-ontology-app-client-id>",
  "iss": "https://login.microsoftonline.com/{tenant-id}/v2.0",
  "oid": "abc123-user-object-id",
  "preferred_username": "jdoe@alasu.edu",
  "name": "Jane Doe",
  "email": "jdoe@alasu.edu",
  "tid": "<tenant-id>",
  "jti": "token-unique-id-for-audit",
  "exp": 1781948717,
  "iat": 1781945117
}
```

A `roles` claim *may* appear if Entra App Roles are configured, but in practice most universities will not have them. **That's fine** — role assignment lives in the ibl.ai platform, not in the token.

### Step 3 — Agent runtime calls the on-premise ontology, forwarding the token

When the agent needs ontology data, it forwards the user's token (and the role header):

```bash
curl -X POST https://ontology.alasu.edu/mcp \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIs..." \
  -H "X-Iblai-Role: FinancialAidCounselor" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call",
       "params":{"name":"get-student-enrollment","arguments":{"student_id":"001234567"}},"id":1}'
```

### Step 4 — Gateway validates the token and resolves permissions

The on-premise gateway receives two things: the Entra ID JWT (proving identity) and `X-Iblai-Role` (declaring role). It:

1. **Validates the JWT signature** against Entra ID's public keys, fetched from `https://login.microsoftonline.com/{tenant-id}/discovery/v2.0/keys` (JWKS).
2. **Checks `aud`** matches the iblai-ontology app registration.
3. **Checks `exp`** — rejects expired tokens.
4. **Checks `iss`** matches the expected tenant.
5. **Reads `X-Iblai-Role`** to get the role.
6. **Resolves what that role can access** via the local `roles.yaml`.

```python
import jwt
from jwt import PyJWKClient

TENANT_ID = "alasu-tenant-id"
CLIENT_ID = "iblai-ontology-app-client-id"
JWKS_URL = f"https://login.microsoftonline.com/{TENANT_ID}/discovery/v2.0/keys"
jwks_client = PyJWKClient(JWKS_URL, cache_keys=True)

def validate_and_resolve(token: str, request_headers: dict) -> dict:
    signing_key = jwks_client.get_signing_key_from_jwt(token)
    claims = jwt.decode(
        token, signing_key.key, algorithms=["RS256"],
        audience=f"api://{CLIENT_ID}",
        issuer=f"https://login.microsoftonline.com/{TENANT_ID}/v2.0",
        options={"require": ["exp", "iss", "aud", "sub"]},
    )

    user_id = claims["oid"]
    email = claims.get("preferred_username") or claims.get("email")

    # Role comes from the platform via X-Iblai-Role. The gateway TRUSTS it because:
    #  1. the request carries a valid Entra ID JWT (identity verified)
    #  2. the request arrives through the firewall from ibl.ai's authorized IPs
    #  3. the ibl.ai platform is the authoritative source for role assignment
    # The gateway only resolves what the role can ACCESS, not who HAS it.
    role = request_headers.get("X-Iblai-Role", "default")
    if role not in ROLES_CONFIG:
        role = "default"

    permissions = resolve_permissions(role)
    return {
        "user_id": user_id, "email": email, "role": role,
        "allowed_toolsets": permissions["mcp_toolsets"],
        "allowed_memory_paths": permissions["memory_paths"],
        "allowed_cache_tables": permissions["cache_tables"],
        "concurrency_limits": permissions.get("concurrency_limits", {}),
        "token_jti": claims.get("jti"),   # for audit logging
    }
```

The `pyjwt[crypto]` dependency is part of the base install (see `pyproject.toml`), so JWT validation needs no extra.

### Step 5 — Scoped response + audit

If the role allows the requested toolset and resource, the tool runs and returns data; otherwise the gateway returns **403**. **Every** access — allowed or denied — is written to `audit_log` with the token's `jti`, so any data access is traceable from the user's login to the specific MCP tool that was called.

### Token-flow summary

```
1. User opens agent chat
2. Agent runtime redirects to Entra ID
3. User authenticates (email + password + MFA)
4. Entra ID issues JWT (oid, email, jti, exp, ...)
5. Agent runtime stores it in the session
6. Agent needs data -> calls on-prem MCP endpoint with Bearer JWT + X-Iblai-Role
7. Gateway validates JWT (signature/aud/exp/iss)
8. Gateway resolves role permissions via roles.yaml
9. Gateway routes to allowed toolset/memory path only
10. Response returned (agent reasons over it, never stores it)
11. Audit entry written (user, role, action, resource, token_jti, timestamp)
```

---

## Where Role Assignment Lives

Deciding *who has what role* lives in the **ibl.ai platform** — not in the Entra ID token, and not in the on-premise server. This is a deliberate design choice with concrete payoffs:

- **No dependency on Entra App Roles** — the university's IT admin does not need to configure them.
- **No local permission tables to manage** on the on-premise server.
- **Centralized user management** in the same platform where agents are configured.
- **The gateway stays simple** — it enforces permissions but never decides who has a role.

The platform sends the role as an `extra_headers` value on the MCP Server Connection:

```
Authorization: Bearer <user's Entra ID token>    (proves identity)
X-Iblai-Role: FinancialAidCounselor               (declares role)
```

Role assignments are managed through the platform's admin dashboard / API (user-scoped connections with `extra_headers`, optionally a CSV import mapping email → role). Full mechanics — including agent-level default roles and the resolution chain — are in [platform-integration.md](/developer/guides/ontology-platform-integration).

### The clean three-way split

| Concern | Where it lives | Managed by |
|---|---|---|
| Who is this user? (identity) | Entra ID JWT | University SSO |
| What role does this user have? (role assignment) | ibl.ai platform (MCP Server Connection `extra_headers`) | Platform admin |
| What can this role access? (role permissions) | On-premise `roles.yaml` | Ontology deployer |

### Entra ID app registration (the minimum)

```
Application name: iblai-ontology
Application (client) ID: <generated>
Redirect URIs: https://base.manager.iblai.app/api/accounts/connected-services/callback/
API permissions: openid, profile, email
Expose an API:
  Application ID URI: api://<client-id>
  Scopes:
    - IblaiOntology.Read   (allows reading ontology data)
```

No App Roles, no group-to-role mapping, no user-to-role assignment in Azure. Register the app, set the redirect URI, done.

### Default role

A request with no (or an unrecognized) `X-Iblai-Role` falls back to a `default` role, so a valid user is never hard-blocked — they get minimal read-only access until the platform admin assigns their proper role:

```yaml
default:
  display_name: "Authenticated User (No Role Assigned)"
  memory_paths:
    - /ontology/courses/_index.md
    - /ontology/courses/by-dept/**
  mcp_toolsets: []
  cache_tables: []
  agents:
    - general-info-agent
```

---

## Role-to-Permission Mapping (`roles.yaml`)

`roles.yaml` lives on the on-premise server and defines **what each role can access**. It does *not* define who has a role. Each role grants `memory_paths` (glob patterns over the text-memory tree), `mcp_toolsets`, `cache_tables`, optional `concurrency_limits`, and the `agents` allowed to use it. The six example roles plus `default`:

```yaml
roles:
  FinancialAidCounselor:
    display_name: "Financial Aid Counselor"
    memory_paths:
      - /ontology/financial-aid/**
      - /ontology/students/**
      - /ontology/enrollment/**
    mcp_toolsets: [financial-aid-tools, enrollment-tools]
    cache_tables: [students, financial_aid, sap_status, isir_data, enrollment, holds]
    agents: [financial-aid-agent]

  AcademicAdvisor:
    display_name: "Academic Advisor"
    memory_paths:
      - /ontology/students/**
      - /ontology/courses/**
      - /ontology/enrollment/**
    mcp_toolsets: [enrollment-tools]
    cache_tables: [students, enrollment, term_summary, courses, course_sections, canvas_activity, advising]
    agents: [advising-agent]

  Registrar:
    display_name: "Registrar Staff"
    memory_paths: [/ontology/**]
    mcp_toolsets: [enrollment-tools, admin-analytics-tools]
    cache_tables: ["*"]
    agents: [registrar-agent, analytics-agent]

  Student:
    display_name: "Student"
    memory_paths:
      - /ontology/students/by-id/${USER_EMPLID}.md
      - /ontology/courses/_index.md
      - /ontology/courses/by-dept/**
    mcp_toolsets: [student-self-service-tools]
    cache_tables: []
    concurrency_limits:
      max_records_per_query: 1
    agents: [student-self-service-agent]

  Executive:
    display_name: "University Executive"
    memory_paths: [/ontology/**]
    mcp_toolsets: ["*"]
    cache_tables: ["*"]
    concurrency_limits:
      max_records_per_query: 1000
      max_export_rows: 500
    agents: [executive-dashboard-agent]

  IblaiOntologyAdmin:
    display_name: "Ontology Administrator"
    memory_paths: [/ontology/_audit/**]
    mcp_toolsets: [admin-analytics-tools]
    cache_tables: [sync_runs, audit_log]
    admin_dashboard: true

  default:
    display_name: "Authenticated User (No Role Assigned)"
    memory_paths:
      - /ontology/courses/_index.md
      - /ontology/courses/by-dept/**
    mcp_toolsets: []
    cache_tables: []
    agents: [general-info-agent]
```

Inspect and validate:

```bash
ontology roles list
ontology roles show Student
ontology roles validate    # checks toolsets/paths actually exist
```

The canonical file ships at `config/roles.yaml`.

---

## `${USER_EMPLID}` Resolution

The `Student` role uses `${USER_EMPLID}` so a student can only ever see their own record. The gateway maps the student's Entra ID `oid` to their PeopleSoft EMPLID via the `identity_map` table (populated during the student sync by matching on email):

```sql
CREATE TABLE identity_map (
    entra_oid       TEXT PRIMARY KEY,   -- from JWT "oid"
    emplid          TEXT NOT NULL,      -- PeopleSoft EMPLID
    email           TEXT,
    full_name       TEXT,
    last_synced_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

When a student with `oid = "abc123"` requests their memory, the gateway:

1. Looks up `abc123` → EMPLID `001234567`.
2. Substitutes `${USER_EMPLID}` in the role's `memory_paths` → `/ontology/students/by-id/001234567.md`.
3. Allows access only to that file.

Even if a student somehow learns another student's ID, the role constraint pins them to their own record.

---

## Why This Design Is Strong

- **No custom identity system** — the university's existing Entra ID does authentication. No new passwords or MFA.
- **No dependency on SSO maturity** — a basic app registration is all that's needed; role assignment is entirely in the platform.
- **Clean separation of concerns** — identity in Entra, role assignment in the platform, role permissions in `roles.yaml`. Each layer does one job.
- **Tokens carry through the whole chain** — the same JWT that authenticated the user is forwarded to the gateway. No re-authentication.
- **End-to-end audit** — the JWT's `jti` appears in both the platform's logs and the on-prem `audit_log`, so any access is traceable from login to tool call.
- **Students see only their own data** — via EMPLID resolution.

---

## Option B — Per-User OAuth Directly on iblai-ontology (Notion Model)

> **Roadmap / alternative.** Option A above is implemented first. Option B can be added later and can coexist with Option A.

An alternative is for the on-premise server itself to handle per-user OAuth, modeled after Notion's hosted MCP server (`mcp.notion.com`). In Notion's approach the MCP server hosts its *own* OAuth flow; any MCP client (Cursor, Claude Code, VS Code, ChatGPT, the ibl.ai platform) connects directly, the user consents, the server issues a per-user token scoped to their permissions, and no intermediary platform is required.

Applied to iblai-ontology, the server runs its own OAuth 2.0 provider (backed by Entra ID for *identity*, but managing its own *authorization* layer):

```
1. MCP client requests https://ontology.alasu.edu/mcp
2. Server responds 401 with an OAuth challenge:
     WWW-Authenticate: Bearer realm="iblai-ontology",
       authorization_uri="https://ontology.alasu.edu/oauth/authorize"
3. Client opens the authorization URL in the browser
4. The server's OAuth flow redirects to Entra ID for identity
5. User authenticates (email + password + MFA)
6. Entra ID redirects back with an authorization code
7. Server exchanges the code, identifies the user (email, oid),
   and looks up their role in its local user_permissions table
8. Server issues its OWN session token to the client:
     { access_token, refresh_token, user, role, scopes }
9. Client stores the token; uses it on every call
10. Server validates its own token per request, resolves role + permissions
```

In this model the on-premise server manages **both** role assignment and role permissions. Role assignment moves on-premise:

```sql
CREATE TABLE user_permissions (
    id           SERIAL PRIMARY KEY,
    email        TEXT NOT NULL,
    entra_oid    TEXT,
    roles        TEXT[] NOT NULL,
    assigned_by  TEXT,
    assigned_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_user_perms_email ON user_permissions(email);
```

An on-prem admin API/dashboard manages assignments (single `POST /admin/api/permissions`, or bulk CSV import). Role *permissions* still come from the same `roles.yaml`.

### Standards: RFC 9728 + RFC 7591

Option B is also how iblai-ontology becomes auto-discoverable by standards-compliant MCP clients:

- **RFC 9728 — OAuth 2.0 Protected Resource Metadata.** The server publishes metadata describing its authorization server and required scopes, so a client can *discover* how to authenticate without manual configuration.
- **RFC 7591 — OAuth 2.0 Dynamic Client Registration.** Clients register themselves with the authorization server programmatically, no pre-shared client IDs.

These are exactly the two RFCs the ibl.ai platform already implements in its `discover_and_register_mcp_oauth_service` flow (see [platform-integration.md](/developer/guides/ontology-platform-integration)) — so an Option B iblai-ontology slots into the platform's existing OAuth machinery, *and* works for any other RFC-compliant MCP client.

### Option A vs. Option B

| Dimension | Option A: ibl.ai Platform (implemented first) | Option B: Notion model (roadmap) |
|---|---|---|
| Role assignment lives in | ibl.ai platform (`extra_headers`) | On-prem `user_permissions` table |
| Role permissions live in | On-prem `roles.yaml` | On-prem `roles.yaml` |
| Who can connect? | Only the ibl.ai platform | Any MCP client (Cursor, Claude Code, VS Code, ChatGPT, custom apps) |
| User auth flow | Platform in-chat OAuth → forwards Entra token | Direct OAuth on iblai-ontology |
| Discovery / registration | Managed by platform | RFC 9728 metadata + RFC 7591 dynamic registration |
| On-prem complexity | Lower (no OAuth server / sessions) | Higher (runs OAuth provider, sessions, token refresh) |
| Works fully air-gapped | No (needs the platform) | Yes (self-contained; local LLM + local client) |
| Audit trail | Split (platform + on-prem) | Fully on-prem |
| Token refresh | Platform handles it | On-prem server handles it |

**When to use which.** Option A is simplest and fits deployments where ibl.ai is the sole consumer — the platform handles all OAuth complexity. Option B makes iblai-ontology a standalone product any MCP client can connect to, and supports fully air-gapped deployments, at the cost of running an OAuth provider on-premise. **Both can coexist:** the gateway can accept platform tokens (via `X-Iblai-Role`) *and* its own self-issued tokens, so ibl.ai agents use the platform path while power users and direct MCP clients use the Notion-style path.

---

## Related

- Component summary: [components/03-identity.md](https://github.com/iblai/ontology/blob/main/docs/components/03-identity.md)
- How tokens arrive and are gated: [components/04-mcp-outbound.md](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md)
- Platform-side role assignment and OAuth: [platform-integration.md](/developer/guides/ontology-platform-integration)
- The `audit_log` and `identity_map` schema: [components/02-knowledge-materialization.md](https://github.com/iblai/ontology/blob/main/docs/components/02-knowledge-materialization.md)
