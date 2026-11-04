# Built-in MCP Servers

Connect an MCP client (Claude Code, Cursor, Claude Desktop, or your own agent) to the ibl.ai platform's built-in [Model Context Protocol](https://modelcontextprotocol.io) servers, and call platform features as tools.

> **Note:** This guide runs against the hosted [iblai.app](https://iblai.app) environment. For a license to run the full ibl.ai platform on your own infrastructure, reach out to our team at [ibl.ai/contact](https://ibl.ai/contact).

---

## Overview

The platform's ASGI service hosts a set of MCP servers under `/mcp/`. Each server groups related tools: agent settings and memory, sessions and projects, chat with a deployed agent, workflows, Canvas course data, and a few account helpers.

- **Transport:** Streamable HTTP. Every server is a single URL that accepts MCP JSON-RPC over `POST`.
- **Authentication:** the same credentials as the REST API. A tool can only do what the caller can already do through the API; RBAC and tenant scoping apply unchanged.
- **Execution:** a tool runs the same view as the matching REST endpoint, inside the platform. The result is the endpoint's JSON response, returned as tool text.

Most tools are **generated from the `/v1` API**: one tool per published `/v1` route, named and documented after that route. A few tools are hand-written for things the REST API does not cover directly, such as chatting with an agent or running a workflow. See [How tools are generated](#how-tools-are-generated).

This is different from two related pages:

- [MCP Servers](/developer/platform/mcp/mcp-servers) covers the `iblai/vibe` skills and the hosted agent-chat server for coding agents.
- [MCP Server Connections](/developer/platform/mcp/mcp-connections) covers registering *external* MCP servers that your agents call during a chat.

---

## Servers

Base URL on the hosted environment: `https://asgi.data.iblai.app`. On a self-hosted deployment, use your ASGI host.

| Server | URL path | What it covers |
| --- | --- | --- |
| Agent create | `/mcp/agent-create/` | Agent settings, public settings, memories, skills |
| Platform | `/mcp/platform/` | Sessions, pinned sessions, projects, disclaimers, notifications, organization metadata, LLMs, apps, audit logs, chat privacy, permission checks |
| Agent chat | `/mcp/agent-chat/` | Send a prompt to one deployed agent and get its reply |
| Workflows | `/mcp/workflows/` | Run a workflow and get its result |
| Analytics | `/mcp/analytics/` | Current user, the user's platforms, user count |
| Canvas course | `/mcp/canvas/course/` | Read-only Canvas LMS course data for platforms with a Canvas configuration |
| Search | `/mcp/search/` | Health check only, until search routes are published in `/v1` |
| User | `/mcp/user/` | Health check only, until user routes are published in `/v1` |

Keep the trailing slash on each URL.

Every server has a `ping` tool that returns the server time. Use it to check that the URL and credentials work.

The workflows server is on by default and is off when the deployment sets `IBL_ENABLE_WORKFLOWS` to false.

---

## Authentication

Send one `Authorization` header on every request. Three schemes are accepted:

```text
Authorization: Api-Token <PLATFORM_API_KEY>
Authorization: Api-Key <PLATFORM_API_KEY>
Authorization: Bearer <ACCESS_TOKEN>
```

**Organization.** Organization and user are never tool arguments.

- With a **Platform API key**, the organization is the key's platform. A different organization cannot be selected.
- With a **user access token**, send the organization key in an `iblai-org` header. A tool that needs an organization and finds none returns:

  ```json
  {"error": "missing_org", "detail": "This tool needs an organization. Use a platform API key or send the iblai-org header."}
  ```

**User.** The acting user is always the authenticated user behind the token. A tool cannot act as another user.

**Agent chat.** The agent-chat server also needs the agent to talk to:

```text
X-Mentor-Unique-Id: <AGENT_UNIQUE_ID>
```

To create a Platform API key, see the [App CLI](/developer/vibe) login flow or your platform admin settings.

---

## Connect a Client

### Claude Code

```bash
claude mcp add --transport http iblai-platform \
  https://asgi.data.iblai.app/mcp/platform/ \
  --header "Authorization: Api-Token $IBLAI_API_KEY"

claude mcp add --transport http iblai-agent-create \
  https://asgi.data.iblai.app/mcp/agent-create/ \
  --header "Authorization: Api-Token $IBLAI_API_KEY"
```

### Cursor and other JSON-configured clients

Add the servers you need to the client's MCP config (for Cursor, `~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "iblai-platform": {
      "url": "https://asgi.data.iblai.app/mcp/platform/",
      "headers": { "Authorization": "Api-Token <PLATFORM_API_KEY>" }
    },
    "iblai-agent-create": {
      "url": "https://asgi.data.iblai.app/mcp/agent-create/",
      "headers": { "Authorization": "Api-Token <PLATFORM_API_KEY>" }
    },
    "iblai-agent-chat": {
      "url": "https://asgi.data.iblai.app/mcp/agent-chat/",
      "headers": {
        "Authorization": "Api-Token <PLATFORM_API_KEY>",
        "X-Mentor-Unique-Id": "<AGENT_UNIQUE_ID>"
      }
    },
    "iblai-workflows": {
      "url": "https://asgi.data.iblai.app/mcp/workflows/",
      "headers": { "Authorization": "Api-Token <PLATFORM_API_KEY>" }
    },
    "iblai-analytics": {
      "url": "https://asgi.data.iblai.app/mcp/analytics/",
      "headers": { "Authorization": "Api-Token <PLATFORM_API_KEY>" }
    },
    "iblai-canvas-course": {
      "url": "https://asgi.data.iblai.app/mcp/canvas/course/",
      "headers": { "Authorization": "Api-Token <PLATFORM_API_KEY>" }
    }
  }
}
```

Each tool's schema is loaded into the client's context, so connect only the servers you use.

### Raw HTTP

Any HTTP client can call a server. Send `Accept: application/json, text/event-stream`; the reply may come back as a server-sent event whose `data:` line holds the JSON-RPC response.

```bash
URL=https://asgi.data.iblai.app/mcp/platform/
H=(-H "Authorization: Api-Token $IBLAI_API_KEY" \
   -H "Content-Type: application/json" \
   -H "Accept: application/json, text/event-stream")

# 1. Initialize
curl -s "${H[@]}" "$URL" -d '{"jsonrpc":"2.0","id":1,"method":"initialize",
  "params":{"protocolVersion":"2025-03-26","capabilities":{},
  "clientInfo":{"name":"curl","version":"0"}}}'

# 2. List tools
curl -s "${H[@]}" "$URL" -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'

# 3. Call a tool
curl -s "${H[@]}" "$URL" -d '{"jsonrpc":"2.0","id":3,"method":"tools/call",
  "params":{"name":"projects_list","arguments":{"limit":5}}}'
```

---

## How Tools Are Generated

The platform publishes its public API under `/v1` through the API facade. Each published route has a stable path, method, and SDK operation name. The built-in MCP servers read the same published route table and register one tool per route.

**Only published `/v1` routes become tools.** Legacy `/api/...` URLs that are not part of `/v1` have no tool. As more routes are published in `/v1`, their tools appear on the matching server with no client change beyond reconnecting.

**Names follow the `/v1` operation.** A tool's name is the route's SDK operation name: the path segments joined with `_`, plus the action.

| `/v1` route | Tool |
| --- | --- |
| `GET /v1/agents/{agent_id}/settings` | `agents_settings_retrieve` |
| `PUT /v1/agents/{agent_id}/settings` | `agents_settings_update` |
| `GET /v1/projects` | `projects_list` |
| `POST /v1/projects` | `projects_create` |
| `PATCH /v1/projects/{id}` | `projects_partial_update` |
| `DELETE /v1/projects/{id}` | `projects_destroy` |

Actions are `list`, `retrieve`, `create`, `update` (PUT), `partial_update` (PATCH) and `destroy`. When two routes would get the same name, the later one gets a numeric suffix (`_2`).

**Arguments come from the route's OpenAPI operation.** Path parameters (such as `agent_id`), query parameters and request-body fields are all flat tool arguments. Read-only response fields are not arguments. The tool description lists the method, the `/v1` path, and each parameter.

**Server by namespace.** The first `/v1` path segment picks the server:

| `/v1` namespace | Server |
| --- | --- |
| `agents` | agent-create |
| `analytics` | analytics |
| `users` | user |
| `search`, `recommendations` | search |
| `workflows` | workflows |
| anything else | platform |

**Streaming routes are skipped.** Chat streaming and WebSocket routes (`/v1/chat`) are not tools. Use the agent-chat server for a single prompt and reply, or the [chat API](/developer) for streaming.

---

## Tool Reference

The lists below match the routes published at the time of writing. Call `tools/list` for the current set.

### Agent create (`/mcp/agent-create/`)

| Tool | Route | Purpose |
| --- | --- | --- |
| `agents_settings_retrieve` | `GET /v1/agents/{agent_id}/settings` | Agent settings |
| `agents_settings_update` | `PUT /v1/agents/{agent_id}/settings` | Update agent settings (display name, LLM, prompts, capabilities, ...) |
| `agents_settings_public_retrieve` | `GET /v1/agents/{agent_id}/settings/public` | Public agent settings |
| `agents_memories_retrieve` | `GET /v1/agents/{agent_id}/memories` | Memories grouped by category |
| `agents_memories_list_retrieve` | `GET /v1/agents/{agent_id}/memories-list` | Memories, paginated |
| `agents_memories_create` | `POST /v1/agents/{agent_id}/memories` | Add a memory (`category_slug`, `content`) |
| `agents_memories_partial_update` | `PATCH /v1/agents/{agent_id}/memories/{memory_id}` | Edit a memory |
| `agents_memories_destroy` | `DELETE /v1/agents/{agent_id}/memories/{memory_id}` | Delete a memory |
| `agents_skills_list` | `GET /v1/agents/{agent_id}/skills` | Skills assigned to the agent |
| `agents_skills_create` | `POST /v1/agents/{agent_id}/skills` | Assign an existing skill (`skill` is the skill id) |
| `ping` | | Health check |

`category_slug` must be an active memory category of that agent. New agents get the default categories, such as `learning_goals` and `knowledge_gaps`.

### Platform (`/mcp/platform/`)

| Tool | Route | Purpose |
| --- | --- | --- |
| `sessions_list` | `GET /v1/sessions` | Sessions in a date range |
| `sessions_create` | `POST /v1/sessions` | Get or create a chat session with an agent (`mentor`) |
| `sessions_update` | `PUT /v1/sessions/{session_id}` | Session title and tools |
| `sessions_recent_list` | `GET /v1/sessions/recent` | Recent sessions |
| `sessions_messages_retrieve` | `GET /v1/sessions/{session_id}/messages` | Messages in a session |
| `sessions_messages_destroy` | `DELETE /v1/sessions/{session_id}/messages` | Delete every message in a session |
| `sessions_sources_list` | `GET /v1/sessions/{session_id}/sources` | Sources retrieved in a session |
| `sessions_guided_prompts_retrieve` | `GET /v1/sessions/{session_id}/guided-prompts` | Suggested follow-up prompts |
| `sessions_pinned_list` | `GET /v1/sessions/pinned` | Pinned sessions |
| `sessions_pinned_create` | `POST /v1/sessions/pinned` | Pin a session (`session_id`) |
| `sessions_pinned_destroy` | `DELETE /v1/sessions/pinned` | Unpin a session (`session_id`) |
| `chat_stop_create` | `POST /v1/chat/stop` | Stop a running generation (`generation_id`) |
| `projects_list` | `GET /v1/projects` | Projects |
| `projects_create` | `POST /v1/projects` | Create a project |
| `projects_retrieve` | `GET /v1/projects/{id}` | One project |
| `projects_update` | `PUT /v1/projects/{id}` | Replace a project |
| `projects_partial_update` | `PATCH /v1/projects/{id}` | Edit a project |
| `projects_destroy` | `DELETE /v1/projects/{id}` | Delete a project |
| `disclaimers_list` | `GET /v1/disclaimers` | Disclaimers |
| `disclaimers_create` | `POST /v1/disclaimers` | Create a disclaimer |
| `disclaimers_retrieve` | `GET /v1/disclaimers/{id}` | One disclaimer |
| `disclaimers_update` | `PUT /v1/disclaimers/{id}` | Replace a disclaimer |
| `disclaimers_partial_update` | `PATCH /v1/disclaimers/{id}` | Edit a disclaimer |
| `notifications_retrieve` | `GET /v1/notifications` | Notifications |
| `notifications_update` | `PUT /v1/notifications` | Change notification status |
| `notifications_count_retrieve` | `GET /v1/notifications/count` | Notification count |
| `organization_metadata_retrieve` | `GET /v1/organization/metadata` | Organization metadata |
| `organization_metadata_update` | `PUT /v1/organization/metadata` | Replace organization metadata |
| `organization_metadata_partial_update` | `PATCH /v1/organization/metadata` | Edit organization metadata |
| `configurations_chat_privacy_retrieve` | `GET /v1/configurations/chat-privacy` | Whether users control chat privacy |
| `configurations_chat_privacy_partial_update` | `PATCH /v1/configurations/chat-privacy` | Change that setting (admins) |
| `configurations_chat_privacy_effective_retrieve` | `GET /v1/configurations/chat-privacy/effective` | Effective chat-privacy mode |
| `llms_list` | `GET /v1/llms` | Available LLMs |
| `features_apps_list` | `GET /v1/features/apps` | Apps the user can access |
| `memsearch_status_retrieve` | `GET /v1/memsearch-status` | Whether memory search is on |
| `audit_logs_list` | `GET /v1/audit-logs` | Audit log entries |
| `rbac_permissions_check_create` | `POST /v1/rbac/permissions/check` | Check the caller's permissions on resources |
| `ping` | | Health check |

### Hand-written tools

| Server | Tool | Purpose |
| --- | --- | --- |
| agent-chat | `get_mentor_response(prompt)` | Send one prompt to the agent in `X-Mentor-Unique-Id` and return its reply |
| workflows | `execute_workflow(workflow_id, input_message, timeout?)` | Run an active workflow and return `response`, `workflow_id` and `session_id` |
| analytics | `get_current_user()` | Username of the authenticated user |
| analytics | `get_user_platforms()` | Platforms the user belongs to and administers |
| analytics | `count_users_in_platform(platform)` | Number of active users in a platform |
| canvas course | 17 read-only tools | `list_courses`, `get_course`, `list_course_users`, `get_user_progress`, `get_bulk_user_progress`, `get_course_settings`, `get_course_activity_stream`, `get_effective_due_dates`, and more |

The Canvas course tools call the Canvas LMS API with the Canvas credentials stored for the caller's platform. Without a Canvas configuration they return `No Canvas configuration found for platform '<org>'`.

---

## Results and Errors

- A tool returns the endpoint's JSON response as text: an object, a list, or a paginated `{count, next, previous, results}` envelope.
- An endpoint error comes back the same way, for example `{"error": "Category not found or inactive"}` or a field validation map. The MCP call itself still succeeds, so check the body.
- A `DELETE` or empty `201` response returns `null`.
- Optional arguments that are not sent are left out of the request. `false`, `0` and empty strings are sent as given.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| HTTP `401` | Missing or invalid `Authorization` header | Check the scheme (`Api-Token`, `Api-Key` or `Bearer`) and the key |
| HTTP `403` with a JSON `RequestId` body before any MCP response | An edge firewall in front of the platform blocks the path | Ask your platform operator to allow `/mcp/<server>/` on the ASGI host |
| HTTP `404`, or `403` on a URL without the trailing slash | Wrong path, or the trailing slash is missing | Use the paths in [Servers](#servers), with the trailing slash |
| `missing_org` | User token without an organization | Send `iblai-org: <org key>`, or use a Platform API key |
| `Could not authenticate mentor via X-Mentor-Unique-Id` | Agent-chat call without a valid agent header | Set `X-Mentor-Unique-Id` to an agent the caller can access |
| A tool you expect is missing | Its route is not published in `/v1` yet | Call `tools/list` again after the route is published |

---

## Related

- [MCP Servers](/developer/platform/mcp/mcp-servers): `iblai/vibe` skills and the hosted agent-chat server
- [MCP Server Connections](/developer/platform/mcp/mcp-connections): register external MCP servers for your agents
- [MCP & OAuth Architecture](/developer/platform/mcp/architecture): backend design of MCP connections and OAuth connectors
- [API Reference](/developer): the `/v1` REST endpoints behind each tool
