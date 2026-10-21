# iblai-api-agent-chat

> Mirrored from [`iblai/vibe`](https://github.com/iblai/vibe) · [`skills/agents/iblai-api-agent-chat/SKILL.md`](https://github.com/iblai/vibe/blob/main/skills/agents/iblai-api-agent-chat/SKILL.md). This page is generated — edit it in the repository, not here.

Enable **live chat** with a deployed agent. Unlike the other `iblai-*` skills
(which call the REST API directly), runtime chat — streamed responses, tool use,
RAG — is served by the hosted **`iblai-api-agent-chat` MCP server**. This skill does
the one-time wiring so your assistant can talk to an agent; the conversation
itself then runs through that server's MCP tools.

## When to use

- You want to *converse with* an agent (not configure it). For configuring agents
  use the other `/iblai-api-agent-*` skills; to *find* an agent's `unique_id` use
  `/iblai-api-search`.

## What you need

- **`IBLAI_API_KEY`** — your Platform API Token (from `.env`; run `/iblai-api-login`
  if it is not set).
- **An agent's `unique_id`** — the agent to chat with. Get one from
  `/iblai-api-search` (each result's `unique_id`), then confirm the choice
  with the user.

## Setup

The server authenticates with two headers: `Authorization: Api-Token <key>` and
`X-Mentor-Unique-Id: <unique_id>` (one agent per connection).

### Claude Code

```bash
claude mcp add iblai-api-agent-chat \
  --transport http https://asgi.data.iblai.app/mcp/agent-chat/ \
  --header "Authorization: Api-Token $IBLAI_API_KEY" \
  --header "X-Mentor-Unique-Id: $MENTOR"
```

### .mcp.json / Claude Desktop / Cursor

Add to the project's `.mcp.json` (or the client's MCP config). Substitute the real
token and `unique_id` — do not commit secrets:

```json
{
  "mcpServers": {
    "iblai-agent-chat": {
      "transport": "streamable-http",
      "url": "https://asgi.data.iblai.app/mcp/agent-chat/",
      "headers": {
        "Authorization": "Api-Token YOUR_API_TOKEN",
        "X-Mentor-Unique-Id": "YOUR_MENTOR_UNIQUE_ID"
      }
    }
  }
}
```

After adding it, the assistant gains the server's chat tools — use those to send
messages and receive the agent's streamed replies.

## Notes

- **One agent per connection** — `X-Mentor-Unique-Id` is fixed per server entry.
  To chat with several agents, add multiple named entries (e.g.
  `iblai-api-agent-chat-tutor`, `iblai-api-agent-chat-advisor`), each with its own
  `unique_id`.
- The token lives in client config / `.env` — never commit it; `.env` and local
  MCP config should stay gitignored.
- This is the only runtime surface among the `iblai-api-*` skills; everything else (configure,
  search, recommend, profile, analytics, admin) is a direct-REST skill.
