# iblai-api-agent-tool

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-api-agent-tool/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-api-agent-tool/SKILL.md). This page is generated — edit it in the repository, not here.

Read and change an agent's tools. The agent's tools are one set, saved
through the agent `settings/` endpoint. On API version 4.467.0 or later most tools are
governed by a `capability_*` flag on the agent's settings: flip the flag
(`/iblai-api-agent-setting`) to turn one capability on or off without touching
the others. `tool_slugs` writes the whole set; use it for a tool with no flag,
such as `mcp`.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`,
  `{mentor}` = the agent's unique id (e.g. `d17dc729-60fd-4363-81a0-f67d9318b03e`).
- **Tool writes** go through the agent settings endpoint —
  **PUT** `…/users/{username}/mentors/{mentor}/settings/` (no PATCH).
- `can_use_tools: false` removes every tool from the agent. Confirm with the
  user first.
- Not connected yet? Run **`/iblai-api-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Tools and flags

| Tool `name` | Flag that governs it |
|---|---|
| Web Search, Web Fetch | `capability_web_search` |
| Image Generation | `capability_image_generation` |
| Video Generation | `capability_video_generation` |
| Code Interpreter | `capability_code_interpreter` |
| Advanced Document Search | `capability_advanced_document_search` |
| Grading | `capability_grading` |
| Lesson Completion | `capability_lesson_completion` |
| Human Support | `capability_human_support` |
| Screen Share | `capability_screen_share` |
| MCP (`mcp`) and any other tool | none: only `tool_slugs` adds or removes it |

`capability_subagents` ("Plan") is a flag with no tool, so it never appears in
a tool list. Course Creation, Google Docs, Google Slides, Google Calendar, Deep
Research and Study Mode are no longer listed. They keep working on agents that
already have them, and `tool_slugs` still accepts their slugs.

## Reads

- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentors/{mentor}/available-tools/?search={q}&ordering={field}`
  — the tools the agent can be given. Each row: `id`, `name`, `slug` (the
  value `tool_slugs` takes), `display_name`, `description`, `category`
  (`{id, name, description}`), `logo`, `metadata`, the `allow_base_agent` /
  `allow_openai_agent` / `allow_openai_assistant` / `allow_google_agent`
  booleans, `is_allowed` and `capability_field`.
  - `is_allowed`: whether the agent's current model can use the tool —
    `true`, `false`, or `null` when no model is known. The schema types it as
    a string, so treat `false` and `"false"` alike. Read the list again after
    changing the agent's model.
  - `capability_field`: the `capability_*` flag that governs the tool, or
    `null` when it has none.
- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentors/{mentor}/settings/`
  — the agent's tools now, as `mentor_tools[]` (read each `slug`), plus
  `can_use_tools`, the `capability_*` flags and `mcp_servers`.

## Writes

- **PUT** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentors/{mentor}/settings/` — replace the agent's tools:
  ```json
  {
    "tool_slugs": "string[] (the whole set)",
    "can_use_tools": "boolean"
  }
  ```
  - **The list replaces the agent's tools.** Every slug left out is removed,
    and every tool-backed flag is set from the list: leave `web-search` out
    and `capability_web_search` turns off. A `capability_*` flag sent in the
    same request wins for its own capability.
  - **Build it from a fresh read.** GET the settings, take the slugs from
    `mentor_tools`, change one, and send the list back. A list built from an
    older read undoes changes saved since.
  - **Empty list:** send `{"tool_slugs": []}` as JSON. Multipart drops an
    empty list, so nothing changes. `"tool_slugs": null` is a `400`.
  - **Every tool in the list is checked again**, not only the one you
    changed. A `400` names what fails: an unknown slug (`No tool found with the
    provided slug: <slug>.`), a tool the model cannot run (`Tool '<name>' is not
    compatible with model '<llm>'.`), `mcp` on a model without MCP (`MCP is not
    supported with the model <llm>`), or `screen-share` while the organization
    has no API key for the AI provider of the agent's voice calls (`… API key
    is required to use screen share with … models.`). The body carries `error`
    and, for these, the original messages in `developer_error`.
  - **`can_use_tools` must be on.** A non-empty list on an agent with
    `can_use_tools: false` is a `400` (`Mentor <name> cannot use tools.`); send
    `"can_use_tools": true` in the same request.
  - **`can_use_tools: false` clears every tool on the agent.** Every
    tool-backed flag turns off and `mcp` is removed; `capability_subagents`
    and `mcp_servers` stay. Turning tools back on restores nothing. Never send
    it to remove one tool. Confirm with the user first.

### MCP on or off

MCP has no flag: it is on while `mcp` is among the agent's tools. To turn it
on, add `mcp` to the slugs from a fresh `mentor_tools` read (with
`"can_use_tools": true` when the agent has it `false`). To turn it off, send
that list without `mcp`, as JSON. The agent's `mcp_servers` stay attached
while MCP is off. Servers, connections and sign-in: `/iblai-api-agent-mcp`.

## Example

Turn MCP on without dropping the agent's other tools:

```bash
S="https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/mentors/$MENTOR/settings/"
AUTH="Authorization: Api-Token $IBLAI_API_KEY"
BODY=$(curl -s "$S" -H "$AUTH" | jq -c '
  {tool_slugs: ([.mentor_tools[].slug] + ["mcp"] | unique)}
  + (if .can_use_tools then {} else {can_use_tools: true} end)')
curl -X PUT "$S" -H "$AUTH" -H "Content-Type: application/json" -d "$BODY"
```

## Notes

- **One capability, one flag.** To turn web search, image generation, Human
  Support or another flagged tool on or off, send its `capability_*` flag
  alone (`/iblai-api-agent-setting`). It needs no read first and cannot drop
  another tool.
- **A chat's own tool list** (`tools` on `POST …/users/{username}/sessions/`
  and `PUT …/users/{username}/sessions/{session_id}/`, see
  `/iblai-api-agent-session`) may name only tools the agent has now. One slug
  the agent no longer has (a capability, Human Support or MCP turned off since
  the chat began) refuses the whole request:
  `400 {"error": "Invalid tool slugs received: human-support, mcp."}`. Drop
  the slugs it names and send again.
- **Open chats.** The agent's flags are a ceiling: turning a capability off
  stops it from the next turn of every open chat.
- Take slugs from `available-tools/` or `mentor_tools`; never guess them.
  Examples: `web-search`, `code-interpreter`, `screen-share`, `human-support`,
  `mcp`.
