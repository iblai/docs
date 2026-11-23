# iblai-api-agent-create

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-api-agent-create/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-api-agent-create/SKILL.md). This page is generated — edit it in the repository, not here.

> ★ **Core family.** **With a screen:** `/iblai-vibe-agent-create` mounts the in-app create flow and `/setup` screen — this skill is the headless twin (same data, exact endpoints).

Create a brand-new agent from a template. This is the step **before**
configuring an agent: it mints a new agent and returns its `unique_id`, which
you then hand to the `/iblai-api-agent-*` skills (settings, llm, prompts,
datasets, …) to configure further. To duplicate an existing agent instead of
starting from a template, use `/iblai-api-agent-setting` (fork).

## Auth & conventions

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token $IBLAI_API_KEY` on every request.
- **Path vars:** `{org}` = `$IBLAI_ORG`, `{username}` = `$IBLAI_USERNAME`.
- Not connected yet? Run **`/iblai-api-login`** first to populate `IBLAI_ORG`,
  `IBLAI_USERNAME`, and `IBLAI_API_KEY`.

## Reads

- **GET** `https://api.iblai.app/dm/v1/agents/settings/field-permissions` —
  which fields of the create call the caller may set. Mounted at the `/dm`
  root (not under `/api/ai-mentor/`). The header `iblai-org: {org}` is
  required only for service accounts; every other caller, a Platform API
  Token included, is scoped to its token's organization, so the header can
  only confirm it (`403` if it names another). Returns one entry
  per create field, `{ "<field>": { "write": true|false } }` — e.g.
  `new_mentor_name`, `display_name`, `description`, `categories`,
  `metadata`, `uploaded_profile_image`, `template_name`,
  `mentor_visibility`, `system_prompt`, `proactive_prompt`,
  `guided_prompt_instructions`, `moderation_system_prompt`. A field left out
  of the answer is writable. `403` = the caller may not create agents at all.
- **GET** `https://api.iblai.app/dm/api/search/orgs/{org}/users/{username}/mentors/` — list existing agents in the org (useful to pick a fork source or confirm names are free).
- **GET** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentor/categories/` — category options (to assign after creation via `/iblai-api-agent-setting`).

## Writes

- **POST** `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/users/{username}/mentor-with-settings/` — create a new agent from a template (returns the created agent with its `unique_id`):
  ```json
  {
    "template_name": "string (required, e.g. ai-mentor)",
    "new_mentor_name": "string (required)",
    "display_name": "string",
    "description": "string",
    "system_prompt": "string",
    "llm_provider": "string"
  }
  ```

  The create call **refuses the whole request (`403`) when it carries a field
  the caller may not write**. Read `field-permissions` first and leave those
  fields out: an omitted optional field keeps the agent's default. The two
  required fields are the exception: a caller barred from `template_name` or
  `new_mentor_name` cannot create agents at all, since leaving either out is
  a `400` ("This field is required.") and sending it is the `403`.

## Example

Create a data-science agent from the default `ai-mentor` template, then capture
its `unique_id` for follow-up configuration:

```bash
curl -X POST \
  "https://api.iblai.app/dm/api/ai-mentor/orgs/$IBLAI_ORG/users/$IBLAI_USERNAME/mentor-with-settings/" \
  -H "Authorization: Api-Token $IBLAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "template_name": "ai-mentor",
    "new_mentor_name": "Data Science Tutor",
    "description": "Helps users with statistics and Python."
  }'
# → response includes "unique_id"; use it as {mentor} in the other skills
```

## Notes

- **Field permissions are per caller.** The organization's RBAC policies
  decide them, so two members can get different answers; read them for the
  caller you are creating as, not once for the organization.

- `template_name` is required, usually `ai-mentor` (the default template on
  most orgs); the create call accepts any template the org exposes.
- Creation is intentionally minimal — set only name + a couple of fields here,
  then drive the rest with the focused skills: `/iblai-api-agent-setting`,
  `/iblai-api-agent-llm`, `/iblai-api-agent-prompt`, `/iblai-api-agent-dataset`,
  `/iblai-api-agent-tool`, etc., using the returned `unique_id` as `{mentor}`.
- To copy an existing agent rather than start from a template, use the **fork**
  endpoint in `/iblai-api-agent-setting`.
- **End-to-end example:** the [`iblai/quickstarts`](https://github.com/iblai/quickstarts)
  Python client wraps these calls — create an agent, open a session, and chat with it
  from a script (see also `/iblai-api-agent-session` for the chat transport).

## Reference material

Doc-sourced material preserved from the developer docs:

- [`references/quickstart.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-api-agent-create/references/quickstart.md) — the [`iblai/quickstarts`](https://github.com/iblai/quickstarts) Python client outlined: env vars and the create-agent → open-session → chat-over-WebSocket flow, with the runnable client (`quickstart.py` + `api.py`) preserved in [`assets/`](https://github.com/iblai/base/tree/main/skills/agents/iblai-api-agent-create/assets/).
- [`references/agent-file-format.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-api-agent-create/references/agent-file-format.md) — the `.iblai` Markdown agent-file format ([`iblai/standard`](https://github.com/iblai/standard) pointer).
