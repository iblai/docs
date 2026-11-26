# iblai-api-agent-setting

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-api-agent-setting/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-api-agent-setting/SKILL.md). This page is generated — edit it in the repository, not here.

> ★ **Core family.** **With a screen:** `/iblai-vibe-agent-setting` mounts the agent Settings tab — this skill is the headless twin (same data, exact endpoints).

Read and write an agent's core settings: identity (name, description, category,
profile image), discovery/visibility, feature flags (anonymous, featured, LTI,
attachments, voice, memory, multi-query RAG, reasoning, prompt caching,
incognito, forkable, additional agents) and the `capability_*` flags — all of
which save through the single `settings/` endpoint — plus forking (copy) and
deletion. Use when configuring an agent's basic identity and capabilities.

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`,
  `{mentor}` = the agent's unique id (e.g. `d17dc729-60fd-4363-81a0-f67d9318b03e`).
- **Settings writes** go through one endpoint —
  **PUT** `…/users/{username}/mentors/{mentor}/settings/` with
  `multipart/form-data` — sending **only the changed field(s)**.
- Not connected yet? Run **`/iblai-api-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Capability flags

On API version 4.467.0 or later, each thing an agent can do in a chat is one boolean on
its settings. Flip one flag to turn one capability on or off; the agent's
other tools are left alone.

| Field | The agent can… | Tool behind it (`name`) |
|---|---|---|
| `capability_web_search` | search the web and read pages people share | Web Search, Web Fetch |
| `capability_image_generation` | create images | Image Generation |
| `capability_video_generation` | create short videos | Video Generation |
| `capability_code_interpreter` | run calculations and work with files such as spreadsheets | Code Interpreter |
| `capability_subagents` | split a big request into tasks for helper agents ("Plan") | none |
| `capability_advanced_document_search` | search its documents again while it writes a reply | Advanced Document Search |
| `capability_grading` | grade work against the agent's rubric | Grading |
| `capability_lesson_completion` | mark a course lesson complete for a signed-in user | Lesson Completion |
| `capability_human_support` | hand the chat to a person (a support ticket) | Human Support |
| `capability_screen_share` | see a user's shared screen during a voice call | Screen Share |

- **Reading.** The GET returns every flag as `true` or `false`.
  `capability_subagents` stays on until it is set to `false`. A flag whose
  tool is switched off platform-wide reads `false`, and sending that `false`
  back changes nothing. `embed_enable_screen_sharing` is a read-only copy of
  `capability_screen_share`. Before API version 4.467.0 the GET returns no flags.
- **Writing.** Send only the flag you change: JSON `true` / `false`, or
  multipart `"true"` / `"false"`. A flag left out, or sent as `null`, stays as
  it is, so an unrelated multipart PUT never turns a capability off.
- **Turning one on** (every flag but `capability_subagents`, which has no
  tool) adds its tool to the agent, so it passes the same checks as a tool.
  Each failure is a `400`:
  - the agent has `can_use_tools: false` → `Mentor <name> cannot use tools.`
    Send `"can_use_tools": true` in the same request. Some other fields of a
    refused request may already be saved, so send the flag on its own;
  - the agent's model cannot use it (not checked for `capability_screen_share`)
    → `Tool '<name>' is not compatible with model '<llm>'.` Read `is_allowed`
    per tool first (`/iblai-api-agent-tool`);
  - the tool is switched off platform-wide → `This capability is not available.`;
  - `capability_screen_share`: the organization has no API key for the AI
    provider of the agent's voice calls → `… API key is required to use screen
    share with … models.`

  A validation `400` carries a readable `error` and the original messages in
  `developer_error`. Match on `developer_error`: `error` drops periods.
- **Turning one off** removes its tool and runs no check. To move the agent to
  a model that cannot run a capability, turn the capability off in the same
  PUT as the new `llm_name`.
- **`can_use_tools: false` clears every tool on the agent.** Every flag but
  `capability_subagents` turns off and the `mcp` tool is removed. Never send it
  to turn one capability off.
- **Flags and `tool_slugs` are one state.** A `tool_slugs` write sets every
  tool-backed flag from its list. In the same request, a flag wins for its own
  capability. See `/iblai-api-agent-tool`.
- **Enhanced document retrieval** is two fields kept in step: send
  `capability_advanced_document_search` and `enable_multi_query_rag` with the
  same value in one PUT. `enable_multi_query_rag` runs several searches for
  each question and works with any model. The flag adds a tool that searches
  again during a reply, and the model must support it; when it cannot, send
  `enable_multi_query_rag` alone.
- **Open chats.** The agent's flag is a ceiling: turning one off stops it from
  the next turn of every open chat. Turning one on reaches new chats only.
- **Grading.** Turning `capability_grading` on creates the agent's grader
  setup. While it is on with an empty rubric, chat turns with the agent fail,
  so add a rubric criterion first (`/iblai-vibe-agent-grader`).
- **Permissions.** Each flag is its own settings field: a caller who may not
  write it gets `403`.

## Reads

- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentor/categories/` — category options.
- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentors/{mentor}/settings/` — load all current field values.
- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/mentors/{mentor}/claw-config/` — detect sandbox wiring (404 = not connected).

## Writes

- **PUT** `…/users/{username}/mentors/{mentor}/settings/` — update settings (`multipart/form-data`, send only changed keys):
  ```json
  {
    "mentor_name": "string",
    "mentor_description": "string",
    "categories": "number[]",
    "uploaded_profile_image": "File",
    "mentor_visibility": "string",
    "allow_anonymous": "boolean",
    "is_featured": "boolean",
    "is_lti_accessible": "boolean",
    "show_attachment": "boolean",
    "show_voice_call": "boolean",
    "show_voice_record": "boolean",
    "enable_claw": "boolean",
    "enable_virtual_machine": "boolean",
    "virtual_machine_egress": "none | registries | public | custom",
    "virtual_machine_network_policy_id": "number | null (required when egress is custom)",
    "virtual_machine_secret_ids": "number[] (egress must be public or custom)",
    "enable_memory_component": "boolean",
    "enable_multi_query_rag": "boolean",
    "show_reasoning": "boolean",
    "enable_prompt_caching": "boolean",
    "disable_chathistory": "boolean",
    "forkable": "boolean",
    "enable_client_tools": "boolean",
    "show_explore_mentors": "boolean",
    "can_use_tools": "boolean (false clears every tool on the agent)",
    "capability_web_search": "boolean | null (null = unchanged)",
    "capability_image_generation": "boolean | null",
    "capability_video_generation": "boolean | null",
    "capability_code_interpreter": "boolean | null",
    "capability_subagents": "boolean | null",
    "capability_advanced_document_search": "boolean | null",
    "capability_grading": "boolean | null",
    "capability_lesson_completion": "boolean | null",
    "capability_human_support": "boolean | null",
    "capability_screen_share": "boolean | null"
  }
  ```
  The `capability_*` rules are under [Capability flags](#capability-flags).
  `mentor_visibility` is one of `viewable_by_tenant_admins` (only org admins,
  the creator and people given access find and chat with it),
  `viewable_by_tenant_students` (every org member) or `viewable_by_anyone`
  (every member, and it is listed in the public catalog for non-members).
  `allow_anonymous` is separate: `true` lets anyone with the link chat without
  signing in, whatever the visibility.
  `show_explore_mentors` shows the additional-agents section on the agent's
  welcome screen ("Show additional agents"). `disable_chathistory` makes every
  conversation with this agent incognito; the Privacy tab’s Incognito policy
  sets it (see `/iblai-vibe-incognito`). `show_reasoning` shows the agent’s
  reasoning steps while it responds; `enable_prompt_caching` caches repeated
  prompt context to cut latency and cost on Anthropic models, and also turns
  off document retrieval, memory and the current date/time in the system
  prompt; `enable_multi_query_rag` runs several document searches for each
  question, kept in step with `capability_advanced_document_search`. The
  embed-widget counterparts of `show_attachment` / `show_voice_record` /
  `show_voice_call` (`embed_show_*`) save through the same PUT — see
  `/iblai-api-agent-embed`.
  `enable_client_tools` lets a chat client send its own tools with a turn and
  run them on the user's machine; off by default — see
  `/iblai-api-agent-session` (`references/client-tools.md`).
  The three `virtual_machine_*` keys configure the agent's Linux VM sandbox;
  the networks and environment variables they reference are managed per
  organization — see `/iblai-vibe-agent-sandbox` ("Virtual machine networks &
  environment variables").
- **POST** `…/users/{username}/mentors/{mentor}/fork/` — copy / duplicate agent:
  ```json
  {
    "new_mentor_name": "string (required)",
    "destination_platform_key": "string (required)",
    "clone_documents": "boolean"
  }
  ```
  A fork copies no tools, so it starts with every tool-backed capability off.
- **DELETE** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/{mentor}/` — delete agent (user-scoped path; no body). Destructive — confirm with the user first.

## Example

Rename an agent and make it featured (only the two changed fields are sent):

```bash
curl -X PUT \
  "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/mentors/$MENTOR/settings/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -F "mentor_name=Joseph (Prod)" \
  -F "is_featured=true"
```

## Notes

- A field left out of the PUT is left unchanged — never resend the whole object.
- `categories` expects numeric category ids from the categories endpoint, not names.
- Deletion and forking are scoped to `{username}`; settings reads/writes use the
  `users/{username}/mentors/{mentor}/` path.
