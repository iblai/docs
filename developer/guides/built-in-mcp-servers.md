# Built-in MCP Servers

Connect an MCP client (Claude Code, Cursor, Claude Desktop, or your own agent) to the ibl.ai platform's built-in [Model Context Protocol](https://modelcontextprotocol.io) servers, and call platform features as tools.

> **Note:** This guide runs against the hosted [iblai.app](https://iblai.app) environment. For a license to run the full ibl.ai platform on your own infrastructure, reach out to our team at [ibl.ai/contact](https://ibl.ai/contact).

---

## Overview

The platform's ASGI service hosts a set of MCP servers under `/mcp/`. Each server groups related tools: agent settings and memory, sessions and projects, chat with a deployed agent, workflows, and a few account helpers.

- **Transport:** Streamable HTTP. Every server is a single URL that accepts MCP JSON-RPC over `POST`.
- **Authentication:** the same credentials as the REST API. A tool can only do what the caller can already do through the API; RBAC and tenant scoping apply unchanged.
- **Execution:** a tool runs the same view as the matching REST endpoint, inside the platform. The result is the endpoint's JSON response, returned as tool text.

Most tools are **generated from the `/v1` API**: one tool per published `/v1` route, named and documented after that route. A few tools are hand-written for things the REST API does not cover directly, such as chatting with an agent or running a workflow. See [How tools are generated](#how-tools-are-generated).

This is different from two related pages:

- [MCP Servers](/developer/platform/mcp/mcp-servers) covers the `iblai/vibe` skills and the hosted agent-chat server for coding agents.
- [MCP Server Connections](/developer/platform/mcp/mcp-connections) covers registering *external* MCP servers that your agents call during a chat.

---

## Servers

The MCP servers are served from `https://asgi.data.iblai.app` under `/mcp/...`.

On a self-hosted deployment, use your ASGI host for MCP.

| Server | URL path | What it covers |
| --- | --- | --- |
| Agent create | `/mcp/agent-create/` | Agent settings, public settings, memories, skills |
| Agent chat | `/mcp/agent-chat/` | Send a prompt to one deployed agent and get its reply |
| Analytics | `/mcp/analytics/` | Current user, the key's platform, user count |
| Search | `/mcp/search/` | Health check only, until search routes are published in `/v1` |
| Platform | `/mcp/platform/` | Sessions, pinned sessions, projects, disclaimers, notifications, organization metadata, LLMs, apps, audit logs, chat privacy, permission checks |
| Workflows | `/mcp/workflows/` | Run a workflow and get its result |
| User | `/mcp/user/` | Health check only, until user routes are published in `/v1` |

Keep the trailing slash on each URL.

Every server except agent chat has a `ping` tool that returns the server time. A `ping` confirms the URL and that the key is valid. It does not check what the key can access: call a read tool, such as `agents_settings_retrieve` for one of your agents, for that.

The workflows server is on by default and is off when the deployment sets `IBL_ENABLE_WORKFLOWS` to false.

---

## Authentication

Authenticate with a **Platform API key**. Send it in the `Authorization` header on every request:

```text
Authorization: Api-Token <PLATFORM_API_KEY>
```

**Organization.** A Platform API key belongs to exactly one platform, and that platform is the organization for every tool call. It cannot be changed per request. Organization and user are never tool arguments.

**Use a key from the platform you work on.** A key is created on the platform you are signed in to at the time. To work with the agents of platform A, create the key while signed in to platform A. A key from platform B reports platform A's agents as not found, even if you are an admin of both.

**User.** The acting user is the key's owner. A tool cannot act as another user.

**Agent chat.** The agent-chat server also needs the agent to talk to. Pass the agent's `unique_id`; the agent must belong to the key's platform:

```text
X-Agent-Unique-Id: <AGENT_UNIQUE_ID>
```

`X-Mentor-Unique-Id` is the older name of this header and still works, so existing configurations need no change. If you send both, they must name the same agent; two different ids are refused.

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
        "X-Agent-Unique-Id": "<AGENT_UNIQUE_ID>"
      }
    },
    "iblai-workflows": {
      "url": "https://asgi.data.iblai.app/mcp/workflows/",
      "headers": { "Authorization": "Api-Token <PLATFORM_API_KEY>" }
    },
    "iblai-analytics": {
      "url": "https://asgi.data.iblai.app/mcp/analytics/",
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

## Example: Chat with an Agent

This walkthrough connects a client to one agent and holds a short conversation with it.

### 1. Get what you need

- A **Platform API key** for the platform the agent belongs to (see [Authentication](#authentication)).
- The agent's **`unique_id`**. It is returned as `unique_id` when the agent is created, and it is the `agent_id` the agent-create tools take.

```bash
export IBLAI_API_KEY=<PLATFORM_API_KEY>
export AGENT_ID=<AGENT_UNIQUE_ID>
```

### 2. Connect the agent-chat server

Each agent-chat connection talks to one agent, named by the `X-Agent-Unique-Id` header. To talk to several agents, add one server entry per agent.

Claude Code:

```bash
claude mcp add --transport http iblai-agent-chat \
  https://asgi.data.iblai.app/mcp/agent-chat/ \
  --header "Authorization: Api-Token $IBLAI_API_KEY" \
  --header "X-Agent-Unique-Id: $AGENT_ID"
```

Cursor and other JSON-configured clients:

```json
{
  "mcpServers": {
    "iblai-agent-chat": {
      "url": "https://asgi.data.iblai.app/mcp/agent-chat/",
      "headers": {
        "Authorization": "Api-Token <PLATFORM_API_KEY>",
        "X-Agent-Unique-Id": "<AGENT_UNIQUE_ID>"
      }
    }
  }
}
```

The server has one tool, `get_mentor_response(prompt)`. In Claude Code, ask for it in plain language, for example "Ask the iblai agent what it can help me with".

### 3. Send messages over raw HTTP

```bash
URL=https://asgi.data.iblai.app/mcp/agent-chat/
H=(-H "Authorization: Api-Token $IBLAI_API_KEY" \
   -H "X-Agent-Unique-Id: $AGENT_ID" \
   -H "Content-Type: application/json" \
   -H "Accept: application/json, text/event-stream")

curl -s "${H[@]}" "$URL" -d '{"jsonrpc":"2.0","id":1,"method":"initialize",
  "params":{"protocolVersion":"2025-03-26","capabilities":{},
  "clientInfo":{"name":"curl","version":"0"}}}'

curl -s "${H[@]}" "$URL" -d '{"jsonrpc":"2.0","id":2,"method":"tools/call",
  "params":{"name":"get_mentor_response",
  "arguments":{"prompt":"Hi! In one sentence, what can you help me with?"}}}'
```

The agent's reply is the text of the result:

```json
{"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text",
  "text":"I can answer questions, explain concepts, brainstorm ideas, write and edit text, and help with coding and problem-solving."}],
  "isError":false}}
```

### 4. Hold a conversation

Each `get_mentor_response` call is a new chat: the agent does not remember earlier calls. To continue a conversation, send the earlier turns with the new message. An MCP client such as Claude Code does this on its own when it writes the prompt; a script has to do it itself.

Without history, the third question has nothing to go on:

```text
User:  My name is Sam and I'm planning a 3-day trip to Lisbon. Suggest one must-see place, in one sentence.
Agent: Sam, visit the Belem Tower, Lisbon's iconic riverside fortress and a UNESCO World Heritage Site.

User:  What is my name, and which city am I visiting?
Agent: I don't know your name or which city you're visiting - you haven't mentioned them in this conversation.
```

With the earlier turns included in each prompt, the agent follows along:

```text
User:  My name is Sam and I'm planning a 3-day trip to Lisbon. Suggest one must-see place, in one sentence.
Agent: Visit Belem Tower, Lisbon's iconic riverside fortress and a must-see landmark.

User:  Great. What should I eat there? One sentence.
Agent: Try a warm pastel de nata, Lisbon's famous custard tart, from a local bakery in Belem.

User:  What is my name, and which city am I visiting?
Agent: Your name is Sam, and you're visiting Lisbon.
```

A minimal Python client that keeps the history:

```python
import json
import os
import urllib.request

URL = "https://asgi.data.iblai.app/mcp/agent-chat/"
HEADERS = {
    "Authorization": f"Api-Token {os.environ['IBLAI_API_KEY']}",
    "X-Agent-Unique-Id": os.environ["AGENT_ID"],
    "Content-Type": "application/json",
    "Accept": "application/json, text/event-stream",
}


def rpc(method, params, id_=1):
    body = json.dumps({"jsonrpc": "2.0", "id": id_, "method": method, "params": params})
    req = urllib.request.Request(URL, data=body.encode(), headers=HEADERS, method="POST")
    text = urllib.request.urlopen(req, timeout=120).read().decode()
    # The reply may be a server-sent event; the JSON is on the "data:" line.
    for line in text.splitlines():
        if line.startswith("data:"):
            text = line[len("data:"):]
    return json.loads(text)


def ask(prompt):
    result = rpc("tools/call", {"name": "get_mentor_response", "arguments": {"prompt": prompt}})
    return result["result"]["content"][0]["text"]


rpc("initialize", {"protocolVersion": "2025-03-26", "capabilities": {},
                   "clientInfo": {"name": "chat-example", "version": "0"}})

history = []
for message in [
    "My name is Sam and I'm planning a 3-day trip to Lisbon. Suggest one must-see place, in one sentence.",
    "Great. What should I eat there? One sentence.",
    "What is my name, and which city am I visiting?",
]:
    prompt = message
    if history:
        prompt = "Conversation so far:\n" + "\n".join(history) + "\n\nUser: " + message
    reply = ask(prompt)
    history += [f"User: {message}", f"Assistant: {reply}"]
    print(f"User:  {message}\nAgent: {reply}\n")
```

Replies come from the agent's own model, prompt and knowledge, so wording varies between runs.

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
| agent-chat | `get_mentor_response(prompt)` | Send one prompt to the agent in `X-Agent-Unique-Id` and return its reply |
| workflows | `execute_workflow(workflow_id, input_message, timeout?)` | Run an active workflow and return `response`, `workflow_id` and `session_id` |
| analytics | `get_current_user()` | Username of the authenticated user |
| analytics | `get_user_platforms()` | The platform the API key belongs to |
| analytics | `count_users_in_platform(platform)` | Number of active users in a platform |

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
| HTTP `401` | Missing or invalid `Authorization` header | Send `Authorization: Api-Token <PLATFORM_API_KEY>` with a valid key |
| HTTP `403` with a JSON `RequestId` body before any MCP response | An edge firewall in front of the ASGI host blocks the path | On a self-hosted deployment, allow `/mcp/<server>/` on your ASGI host's edge firewall |
| HTTP `404`, or `403` on a URL without the trailing slash | Wrong path, or the trailing slash is missing | Use the paths in [Servers](#servers), with the trailing slash |
| `Could not authenticate the agent via X-Agent-Unique-Id` | The header is missing, is not the agent's `unique_id`, names an agent outside the key's platform, or `X-Agent-Unique-Id` and `X-Mentor-Unique-Id` name different agents | Send the agent's `unique_id`, with a key created on that agent's platform |
| `Mentor <id> not found.` on an agent-create tool | The agent is not on the key's platform | Create the key on the agent's platform (see [Authentication](#authentication)); `get_user_platforms` shows which platform the key belongs to |
| A tool you expect is missing | Its route is not published in `/v1` yet | Call `tools/list` again after the route is published |

---

## Related

- [MCP Servers](/developer/platform/mcp/mcp-servers): `iblai/vibe` skills and the hosted agent-chat server
- [MCP Server Connections](/developer/platform/mcp/mcp-connections): register external MCP servers for your agents
- [MCP & OAuth Architecture](/developer/platform/mcp/architecture): backend design of MCP connections and OAuth connectors
- [API Reference](/developer): the `/v1` REST endpoints behind each tool
