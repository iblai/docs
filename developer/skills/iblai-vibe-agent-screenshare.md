# /iblai-vibe-agent-screenshare

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-vibe-agent-screenshare/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-vibe-agent-screenshare/SKILL.md). This page is generated — edit it in the repository, not here.

Add the agent **Screen tab** -- how the agent behaves when a user shares their
screen during a voice call. **Scope: per agent.** The **Screen Sharing**
switch saves the agent's `capability_screen_share` flag, which is what puts
the screen-share button in the chat. The two prompt cards live on the agent's
call configuration (one row per agent): **Instructions During Screen
Sharing** (the system prompt used while a screen is shared) and **What the
Agent Says When Screen Sharing Starts** (the proactive opener). Both apply to
every user who calls that agent. This is one tab in the agent-settings family
indexed by `/iblai-vibe-agent`; it shares the `AgentSettingsProvider` wrapper
with every other tab.

**Prompts** -- screen sharing on, both prompts set. Edits stay local until
**Save**.

![Screen tab -- prompts](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-screenshare/iblai-vibe-agent-screenshare-1-prompts.png)

**Edit dialog** -- **Edit** on either card opens the shared rich-text prompt
editor (`EditPromptModal`, the same one the Prompts tab uses). Its **Save**
writes the text back to the card; the tab's **Save** sends it.

![Screen tab -- edit prompt dialog](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-screenshare/iblai-vibe-agent-screenshare-2-edit.png)

**Off** -- with the switch off the cards stay visible but grayed out and
inert, and the header adds "Turn it on to let people share their screen
during a call and edit these prompts." The switch saves immediately and
toasts "Screen Sharing turned on" / "Screen Sharing turned off". An agent
that has only the call configuration's `enable_video` on (all the old switch
wrote) shows off: the chat offers screen sharing only while the flag is on.

![Screen tab -- screen sharing off](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-screenshare/iblai-vibe-agent-screenshare-3-off.png)

> **Common setup (brand, conventions, env files, verification):** see [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/docs/skill-setup.md).

## Prerequisites

- Auth set up (`/iblai-vibe-auth`, or vibe-starter).
- `AgentSettingsProvider` wraps the route (`/iblai-vibe-agent` §1).
- `@iblai/iblai-js@^3.1.0` (resolves `@iblai/web-containers` 2.1.0, where
  the switch saves `capability_screen_share`). Check with
  `pnpm why @iblai/web-containers`.
- A platform that returns `capability_screen_share` on the agent's settings
  (API version 4.467.0 or later). On an older one, or for a person who may
  not read the flag, the switch reads and writes the call configuration's
  `enable_video`, as it did before.
- A real agent UUID. Ask the user; never invent one.
- Voice calls on for the agent (the Voice tab's **Calls** sub-tab,
  `/iblai-vibe-agent-voice`): screen sharing happens inside a voice call, and
  always in **Live Conversation** mode.
- An org admin to save the prompts: call configurations have no agent-level
  permission, so for an agent editor **Save** shows "Couldn't save screen
  sharing prompts". Turning the switch off also writes the call
  configuration while its `enable_video` is on (see How it saves).

## Step 1: Check Environment

Look for `iblai.env` in the project root with `PLATFORM`, `DOMAIN`, and
`TOKEN`. If it is missing, tell the user:
"You need an `iblai.env` with your platform configuration. Download the
template and fill in your values:
`curl -o iblai.env https://raw.githubusercontent.com/iblai/base/refs/heads/main/iblai.env`"

## Step 2: Mount `AgentScreenShareTab`

```tsx
// app/(app)/agents/[mentorId]/screenshare/page.tsx
"use client";

import { AgentScreenShareTab } from "@iblai/iblai-js/web-containers/next";

export default function AgentScreenSharePage() {
  return (
    <div className="flex h-full flex-col bg-white">
      <AgentScreenShareTab />
    </div>
  );
}
```

The tab reads `tenantKey`, `mentorId`, and `username` from
`AgentSettingsProvider` and does its own fetching and saving. No props are
required.

To render the prompts differently on the cards (for example as Markdown with
the renderer you already pass to `AgentPromptsTab`), pass
`renderPromptContent`:

```tsx
import { AgentScreenShareTab } from "@iblai/iblai-js/web-containers/next";

<AgentScreenShareTab
  renderPromptContent={(content) => (
    <p className="text-sm whitespace-pre-wrap text-gray-700">{content}</p>
  )}
/>;
```

## Step 3: Customize Labels (Optional)

The tab renders with the default agent-facing copy
(`AGENT_SCREENSHARE_TAB_LABELS`, localized through the SDK's i18n). Pass a
partial `labels` object to change any string:

```tsx
import { AgentScreenShareTab } from "@iblai/iblai-js/web-containers/next";

<AgentScreenShareTab
  labels={{
    header: { title: "Screen sharing" },
    fields: { proactivePrompt: { label: "Opening line" } },
  }}
/>;
```

Label groups (`ScreenShareTabLabels`): `header` (`title`, `description`),
`capability` (`title` -- the switch's accessible name, "Screen Sharing";
`description`, `offHint`), `disabledHint`, `fields.systemPrompt` and
`fields.proactivePrompt` (`label`, `placeholder` each), `saveButton`,
`savingButton`, `toasts` (`saved`, `error`: the prompts' **Save**, and the
switch without the flag), and `errors` (`turnOff`: shown under the banner
when turning the switch off can't clear `enable_video`). The switch's
"turned on" / "turned off" toasts, its lock reason and a refused flag save's
sentence come from the SDK's catalogue, not from `labels`.

## Step 4: Use MCP Tools for Customization

```
get_component_info("AgentScreenShareTab")
get_component_info("AgentSettingsProvider")
```

## `<AgentScreenShareTab>` Props

Import from `@iblai/iblai-js/web-containers/next`.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `labels` | `DeepPartial<ScreenShareTabLabels>` | No | Override user-visible strings |
| `renderPromptContent` | `(content: string) => ReactNode` | No | Render each prompt as rich content. Defaults to plain text |
| `tenantKey` | `string` | No | Identity override. Falls back to `AgentSettingsProvider` |
| `mentorId` | `string` | No | Identity override (agent UUID). Falls back to `AgentSettingsProvider` |
| `username` | `string` | No | Identity override. Falls back to `AgentSettingsProvider` |

## Related Exports

From `@iblai/iblai-js/web-containers/next`:

- `AGENT_SCREENSHARE_TAB_LABELS` -- the default label bundle.
- `ScreenShareTabLabels`, `AgentScreenShareTabProps` -- types.

## How it saves

Paths are under `/api/ai-mentor/orgs/{org}/users/{username}/`.

| Action | Request |
|---|---|
| Load | The agent's settings embed `call_configuration`; when absent the tab lists `call-configurations/?mentor=<uuid>` |
| Switch on | `PUT mentors/{uuid}/settings/` with `capability_screen_share: true` only (plus `can_use_tools: true` when the agent could not use tools yet; `false` is never sent) |
| Switch off | While the call configuration's `enable_video` is on, first `PATCH call-configurations/{id}/` with `enable_video: false` (calls keep sharing video while either one is on); then the same `PUT` with `capability_screen_share: false` |
| Switch, without the flag (an older platform, or a person who may not read it) | `PATCH call-configurations/{id}/` with `enable_video`; with no row yet, `POST call-configurations/` (`mentor`, `mode: "realtime"`, `language: "en"`, `enable_video`) |
| **Save** | `PATCH call-configurations/{id}/` (or `POST` when no row exists) with `screensharing_system_prompt` and `screensharing_proactive_prompt` |

If clearing `enable_video` fails, the switch stays on and says "Couldn't turn
off Screen Sharing for voice calls. Try again." under the banner. A refused
flag save turns the switch back and shows one sentence there, never a toast.
The switch is locked, with "You don't have permission to change this.", when
the person may not change the flag, or, while it is off on an agent that
cannot use tools yet, may not change `can_use_tools`. The agent's model never
locks it. **Save** stays disabled until a prompt changes.

## Platform data

| Hook | Purpose |
|---|---|
| `useGetMentorSettingsQuery` | Agent settings: `capability_screen_share` and the embedded `call_configuration` |
| `useEditMentorMutation` | Save `capability_screen_share` |
| `useGetCallConfigurationsQuery` | Fallback list when settings carry no call configuration |
| `useCreateCallConfigurationMutation`, `useUpdateCallConfigurationMutation` | Create / patch the row |

REST twin: [`/iblai-api-agent-voice`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-api-agent-voice/SKILL.md)
(call configurations: `enable_video`, `screensharing_system_prompt`,
`screensharing_proactive_prompt`).

## Step 5: Verify

Run `/iblai-vibe-ops-test` before telling the user the work is ready, and give
every page, control and state this skill added its own checkpoint — one Playwright
test titled `<ID> · …` in `e2e/journeys/`, listed in `e2e/coverage.json`;
`pnpm e2e:coverage` must pass:

1. `pnpm build` -- must pass with zero errors.
2. `pnpm test` -- vitest must pass.
3. Touch test `/agents/<uuid>/screenshare` in the browser as `/iblai-vibe-ops-test` says:
   the switch and both prompt cards render; turn the switch on ("Screen Sharing turned
   on"); edit the opener, **Save**, reload -- the text persists; read the console; check
   the phone width.

## Important Notes

- **Per agent, not per user**: one call configuration per agent; every caller
  gets the same prompts.
- **Default system prompt**: a new call configuration starts with the
  platform's screen-guidance prompt, so the first card is rarely empty.
- **Live conversation only**: screen sharing ignores the call style set on
  the Voice tab's **Calls** sub-tab and always streams.
- **A missing key**: when voice calls need an API key from their AI provider
  and none is saved, turning Screen Sharing on is refused with "Screen
  Sharing needs an API key from the AI provider this agent uses on voice
  calls. Add it in Secrets in your organization settings, then turn it on
  again." While Screen Sharing is on without that key, other capability
  switches and model changes can be refused for the same reason, and say so.
- **Shared provider**: mount `AgentSettingsProvider` once at the layout
  level (`/iblai-vibe-agent` §1); do not wrap each tab.
- **Peer deps**: `sonner` and `@iblai/iblai-web-mentor`
  (`pnpm add sonner @iblai/iblai-web-mentor`).
- **Brand guidelines**: [BRAND.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/BRAND.md)
