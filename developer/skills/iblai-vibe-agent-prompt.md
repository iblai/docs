# /iblai-vibe-agent-prompt

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-vibe-agent-prompt/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-vibe-agent-prompt/SKILL.md). This page is generated — edit it in the repository, not here.

Add the agent **Prompts tab** -- a full-width **System Prompt** card, then
**Proactive Prompt** and **Guided Prompt** side by side (stacked on
phones), each with **Edit** and **Copy** buttons; Proactive and Guided
each carry an Active/Inactive switch. Below them, a **Suggested Prompts**
section with add and edit modals. This is one tab in the wider
agent-settings family. All tabs share the same `AgentSettingsProvider`
wrapper.

![Prompts Tab](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-prompt/iblai-vibe-agent-prompt.png)

> **Common setup (brand, conventions, env files, verification):** see [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/docs/skill-setup.md).

## Prerequisites

- Auth must be set up first (`/iblai-vibe-auth`)
- MCP server + skills configured (`@iblai/mcp` in `.mcp.json`)
- `AgentSettingsProvider` must wrap the route (see `/iblai-vibe-agent-setting`
  Step 2 if not already set up)
- Ask the user for a real `mentorId` (agent UUID). Do NOT invent one.

## Step 1: Check Environment

Before proceeding, check for an `iblai.env` in the project root. Look for
`PLATFORM`, `DOMAIN`, and `TOKEN` variables. If the file does not exist or
is missing these variables, tell the user:
"You need an `iblai.env` with your platform configuration. Download the
template and fill in your values:
`curl -o iblai.env https://raw.githubusercontent.com/iblai/base/refs/heads/main/iblai.env`"

## Step 2: Mount `AgentPromptsTab`

```tsx
// app/(app)/agents/[mentorId]/prompts/page.tsx
"use client";

import { AgentPromptsTab } from "@iblai/iblai-js/web-containers/next";

export default function AgentPromptsPage() {
  return (
    <div className="flex h-full flex-col bg-white">
      <AgentPromptsTab />
    </div>
  );
}
```

### With Markdown rendering

```tsx
import ReactMarkdown from "react-markdown";

<AgentPromptsTab
  renderPromptContent={(content) => <ReactMarkdown>{content}</ReactMarkdown>}
/>;
```

## Step 3: Customize Labels (Optional)

```tsx
import { AgentPromptsTab } from "@iblai/iblai-js/web-containers/next";

<AgentPromptsTab
  labels={{
    header: { title: "Mentor prompts" },
  }}
/>;
```

The label bundle has `header`, `prompts.system`, `prompts.proactive`,
`prompts.guided`, `prompts.suggested`, `actions` and `toasts`. There is
no `prompts.study` key any more: an object-literal override that still
sets one fails to typecheck.

## Step 4: Use MCP Tools for Customization

```
get_component_info("AgentPromptsTab")
get_component_info("AgentSettingsProvider")
```

## `<AgentPromptsTab>` Props

Import from `@iblai/iblai-js/web-containers/next`.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `labels` | `DeepPartial<PromptsTabLabels>` | No | Override user-visible strings |
| `renderPromptContent` | `(content: string) => ReactNode` | No | Render prompt text as rich content (e.g., Markdown). Defaults to plain text |

## Related Exports

From `@iblai/iblai-js/web-containers/next`:

- `AGENT_PROMPTS_TAB_LABELS` -- the default agent-facing label bundle.
- `PromptsTabLabels` -- type for the full label bundle.
- `GreetingMethod` -- enum of the `greeting_method` values the Proactive
  Prompt switch writes: `GreetingMethod.PROACTIVE_PROMPT` is
  `"proactive_prompt"`, `GreetingMethod.PROACTIVE_RESPONSE` is
  `"proactive_response"` (the wire values are lower case).

## What Each Card Saves

The three prompt cards write to the agent's settings
(`useEditMentorMutation`), one field per save; a suggested prompt is a
record of its own.

| Card | **Edit** saves | Switch saves |
|------|----------------|--------------|
| System Prompt | `system_prompt` | -- (no switch) |
| Proactive Prompt | `proactive_prompt` | `greeting_method`: `proactive_prompt` when Active (the agent answers this prompt to open the chat), `proactive_response` when Inactive (the agent's stored greeting text, no LLM call; this tab does not edit that text) |
| Guided Prompt | `guided_prompt_instructions` (how the suggestions are written) | `enable_guided_prompts` (the chat suggests next prompts, written by the LLM from the conversation) |
| Suggested Prompts | the prompt text, its category and visibility (**Add New Prompt** creates one, **Edit** changes it; delete is not in the tab, see `/iblai-api-agent-prompt`) | -- |

## Step 5: Verify

Run `/iblai-vibe-ops-test` before telling the user the work is ready, and give
every page, control and state this skill added its own checkpoint — one Playwright
test titled `<ID> · …` in `e2e/journeys/`, listed in `e2e/coverage.json`;
`pnpm e2e:coverage` must pass:

1. `pnpm build` -- must pass with zero errors
2. `pnpm test` -- vitest must pass
3. Touch test `/agents/<id>/prompts` in the browser as `/iblai-vibe-ops-test` says: open it,
   use every control this skill added, read the console, check the phone width.

## Important Notes

- **Redux store**: Must include `mentorReducer` and `mentorMiddleware`
- **`initializeDataLayer()`**: 5 args (v1.2+)
- **`@reduxjs/toolkit`**: Deduplicated via webpack aliases in `next.config.ts`
- **Peer deps**: `sonner` and `@iblai/iblai-web-mentor` must be installed
  (`pnpm add sonner @iblai/iblai-web-mentor`)
- **Shared provider**: `AgentSettingsProvider` must wrap the route at a
  layout level. See `/iblai-vibe-agent-setting` Step 2 for the full snippet.
- **Brand guidelines**: [BRAND.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/BRAND.md)
