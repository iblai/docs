# /iblai-vibe-agent-tool

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-vibe-agent-tool/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-vibe-agent-tool/SKILL.md). This page is generated — edit it in the repository, not here.

Since `@iblai/iblai-js` 3.0.0 (`@iblai/web-containers` 2.0.0) the agent
settings have no **Tools** tab. Each thing an agent can do has its own switch,
in the tab it belongs to. A click saves that switch's setting of the agent at
once, with no Save button and no list of tools to rewrite. **Scope: per
agent.** Every user who chats with the agent gets what is switched on.

This skill builds nothing. It says which tab holds which switch, how the
switches behave, what the chat shows, and how to remove an old Tools tab.

> **Common setup (brand, conventions, env files, verification):** see [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/docs/skill-setup.md).

## Where each switch lives

Every tab below reads the agent from `AgentSettingsProvider`
(`/iblai-vibe-agent` §1). Mount the tab, and its switch comes with it.

| Capability | Tab, and where in it | A click saves | Skill |
|---|---|---|---|
| Web Search | Settings › Capabilities › **Agent Capabilities** | `capability_web_search` | `/iblai-vibe-agent-setting` |
| Image Generation | Settings › Capabilities › Agent Capabilities | `capability_image_generation` | `/iblai-vibe-agent-setting` |
| Video Generation | Settings › Capabilities › Agent Capabilities | `capability_video_generation` | `/iblai-vibe-agent-setting` |
| Code Interpreter | Settings › Capabilities › Agent Capabilities | `capability_code_interpreter` | `/iblai-vibe-agent-setting` |
| Plan (splits a big request into tasks for helper agents) | Settings › Capabilities › Agent Capabilities | `capability_subagents` | `/iblai-vibe-agent-setting` |
| Enhanced Document Retrieval | Datasets, the box under the info box | `capability_advanced_document_search` and `enable_multi_query_rag`, in one request; only `enable_multi_query_rag` while the flag is off and the agent's model cannot run the search tool (the box then says "With this model, your agent can't search again while it works on a reply.") | `/iblai-vibe-agent-dataset` |
| Grading | Grader, the switch at the top | `capability_grading` | `/iblai-vibe-agent-grader` |
| Lesson Completion | Grader, the row under the Grading switch | `capability_lesson_completion` | `/iblai-vibe-agent-grader` |
| Human Support | Support, the **Human Support** switch | `capability_human_support` | `/iblai-vibe-agent-support` |
| Screen Sharing | Screen, the switch at the top | `capability_screen_share` | `/iblai-vibe-agent-screenshare` |
| MCP | MCP, the switch at the top | adds or removes the `mcp` tool in `tool_slugs` (MCP has no flag) | `/iblai-vibe-agent-mcp` |
| Voice Calls | Voice › Calls, one switch for In-App Chat and one for Embed Widget | `show_voice_call`, `embed_show_voice_call` | `/iblai-vibe-agent-voice` |
| Dictation | Voice › Dictation, one switch per surface | `show_voice_record`, `embed_show_voice_record` | `/iblai-vibe-agent-voice` |
| Smart Document Retrieval | Voice › Calls, above the Voice picker; saved by that sub-tab's **Save** | the call configuration's `use_function_calling_for_rag` | `/iblai-vibe-agent-voice` |

Two places hide their switch on purpose. `AgentDatasetsTab` and `McpTab` leave
it out in picker mode (`onSelect`) and when you pass
`showCapabilitySwitch={false}`, as the SDK's Create Agent wizard does while
the agent is still being set up.

**A tool with no switch of its own** (Google Docs, for example) has no
control in the agent settings any more. Set it over REST (`tool_slugs`,
[`/iblai-api-agent-tool`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-api-agent-tool/SKILL.md))
or with `useToggleTools` (below).

## How a switch behaves

The capability switches (every `capability_*` row above) share one behaviour:

- **Saves on the click** through `useEditMentorMutation`, the agent's settings
  `PUT`, with only its own flag (Enhanced Document Retrieval adds
  `enable_multi_query_rag`). When a capability with a tool behind it is
  turned on for an agent that could not use tools yet, the request also sends
  `can_use_tools: true`. Nothing sends `can_use_tools: false`: the platform
  would clear every tool on the agent.
- **Confirms with a toast**: "Web Search turned on", "Web Search turned off".
- **Locked, with the reason under it**, when the user may not change the flag
  ("You don't have permission to change this.", from
  `permissions.field[flag].write === false`), may not let the agent use tools
  while turning it on needs that, or the agent's model cannot use the
  capability ("Doesn't work with the model this agent uses. Pick another model
  in the LLM tab to turn it on."). The model check reads the `capability_field`
  and `is_allowed` columns of the agent's tool list (`useGetToolsQuery`), read
  again each time a tab with a switch opens and right after the LLM tab saves a
  model. Enhanced Document Retrieval is the exception: while its flag is off
  and the model cannot run the search tool, the model does not lock the
  switch, which then saves `enable_multi_query_rag` alone and needs write
  access to that field only.
- **A refused save turns the switch back** and shows one sentence under it,
  never a toast and never the platform's own text. For example "Web Search
  doesn't work with the model this agent uses. Pick another model in the LLM
  tab, then try again." or "Couldn't update Web Search. Try again."
- **Renders nothing** when the platform does not return the flag or the user
  may not read it, except Grading, whose switch then stays off and locked, and
  Screen Sharing, whose switch then reads and writes the call configuration's
  `enable_video` (see Compatibility).

Two tabs add a rule of their own. Grading stays off and locked ("Add at least
one rubric criterion first, then turn Grading on.") until the rubric has a
criterion. Turning Screen Sharing off also clears the call configuration's
`enable_video` when it was on. Their skills have the details.

The Voice switches save their own field on the click the same way, and a
refused save turns the switch back with a sentence under its box. The MCP
switch saves `tool_slugs` instead of a flag; `/iblai-vibe-agent-mcp` has the
details.

The LLM tab is where a capability can get in the way. Picking a model that
cannot run a capability the agent has on is refused by the platform, and the
toast says which one and where to turn it off: "Grading is on, and the model
you picked can't use it. Turn it off in the Grader tab first, or pick a
different model." (`/iblai-vibe-agent-llm`).

## In the chat

`Chat` (`/iblai-vibe-agent-chat`) follows the switches with no host code:

- The `+` menu's Tools (**Web Search**, **Code**, **Image**) and the
  **Screen Sharing** button follow the agent's Web Search, Code Interpreter,
  Image Generation and Screen Sharing flags. Where the platform returns no
  flag, the chat falls back to the agent's tool list.
- The `+` menu's MCP row is hidden while MCP is off for the agent.
- A conversation's tools keep only the tools the agent still has, so a
  capability switched off after the chat began does not make the chat's other
  tool toggles fail.

To read the flags in your own UI, use `useMentorSettings`. Its
`data.capabilities` holds each `capability_*` flag the platform returned (a
flag it did not return is absent):

```tsx
"use client";

import { useMentorSettings } from "@iblai/iblai-js/web-utils";

export function WebSearchBadge({ agentId, orgKey }: { agentId: string; orgKey: string }) {
  const { data } = useMentorSettings({ mentorId: agentId, tenantKey: orgKey });
  if (!data.capabilities.capability_web_search) return null;
  return <span className="text-xs text-gray-600">Web search on</span>;
}
```

## Compatibility

The `capability_*` switches need the platform at API version 4.467.0 or later. An older
platform returns no `capability_*` flags in the agent's settings, and then:

- the Agent Capabilities block, the Datasets box, the Lesson Completion row
  and the Human Support switch render nothing;
- the Grading switch stays off and locked;
- the Screen switch keeps writing the call configuration's `enable_video`, as
  before.

The MCP and Voice switches do not depend on the platform version. To check a
platform, read one agent's settings (`useGetMentorSettingsQuery`, or the REST
call in `/iblai-api-agent-setting`): when `capability_web_search` is in the
response, the switches show. On an older platform, switch tools over REST
(`/iblai-api-agent-tool`) until it is upgraded.

## Migrate a host that mounted `AgentToolsTab`

`@iblai/iblai-js/web-containers/next` no longer exports `AgentToolsTab`,
`AGENT_TOOLS_TAB_LABELS`, `AgentToolsTabProps`, `ToolsTabLabels` or
`GRADING_TOOL_NAME`, so `pnpm build` fails until they are gone. Nothing
replaces the tab:

1. Delete the Tools tab page (for example
   `app/(app)/agents/[mentorId]/tools/page.tsx`), its menu entry or tab link,
   and every link to it. If old links may still arrive, redirect them to the
   Settings tab (a `redirects()` entry in `next.config.ts`).
2. Mount the tabs that hold the switches you need (table above). An app that
   already mounts Settings and Datasets has the five chat capabilities and
   Enhanced Document Retrieval with no change.
3. Code that looked up the Grading tool by `GRADING_TOOL_NAME` reads
   `capability_grading` from the agent's settings instead.
4. A host that passes a full `SettingsTabLabels` object removes the keys 3.0
   dropped (`/iblai-vibe-agent-setting`, Step 4).

`useToggleTools({ activeMentorId, tenantKey, username })` stays exported from
`@iblai/iblai-js/web-containers/next` for the workflow builder's Agent step.
`toggleTools(slug)` adds or removes one slug: it reads the agent's tools inside
each toggle (and sends nothing when that read fails), then saves the new
`tool_slugs` with `can_use_tools: true`, or sends `{ tool_slugs: [] }` as JSON
when the last tool goes. Its `tools` argument is deprecated and ignored.

## Platform data

| Hook / call | Purpose |
|---|---|
| `useGetMentorSettingsQuery` | The agent's `capability_*` flags, `mentor_tools` and `permissions` |
| `useEditMentorMutation` | `PUT …/mentors/{uuid}/settings/` with the switch's own fields (plus `can_use_tools: true` when needed) |
| `useGetToolsQuery` | The tools available to the agent (`available-tools/`), with `capability_field` and `is_allowed` per row: which capabilities its model can use |
| `useMentorSettings` (`/web-utils`) | `data.capabilities` for the chat and your own UI |

REST: [`/iblai-api-agent-setting`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-api-agent-setting/SKILL.md)
(the flags are agent settings) and
[`/iblai-api-agent-tool`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-api-agent-tool/SKILL.md)
(`tool_slugs`).

## Verify

Run `/iblai-vibe-ops-test` before telling the user the work is ready, and give
every page, control and state you changed its own checkpoint — one Playwright
test titled `<ID> · …` in `e2e/journeys/`, listed in `e2e/coverage.json`;
`pnpm e2e:coverage` must pass:

1. `pnpm build` -- passes, with no import of the removed names left.
2. `pnpm test` -- vitest passes.
3. Touch test in the browser as `/iblai-vibe-ops-test` says: no menu entry
   or link leads to a Tools tab, and an old `/agents/<uuid>/tools` route is
   gone or redirects to Settings; in `/agents/<uuid>/settings` › Capabilities turn
   **Web Search** on, see "Web Search turned on", reload -- it stays on; in
   the chat the `+` menu's Tools lists Web Search; turn it off again; read the
   console; check the phone width.

## Related skills

- `/iblai-vibe-agent` -- the family index and the provider every tab needs.
- `/iblai-vibe-agent-setting`, `/iblai-vibe-agent-dataset`,
  `/iblai-vibe-agent-grader`, `/iblai-vibe-agent-support`,
  `/iblai-vibe-agent-screenshare`, `/iblai-vibe-agent-mcp`,
  `/iblai-vibe-agent-voice` -- the tabs that hold the switches.
- `/iblai-vibe-agent-llm` -- the model decides which capabilities can be on.
- `/iblai-vibe-workflow` -- the workflow builder, whose Agent step uses
  `useToggleTools`.
