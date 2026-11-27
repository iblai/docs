# /iblai-vibe-agent-voice

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-vibe-agent-voice/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-vibe-agent-voice/SKILL.md). This page is generated — edit it in the repository, not here.

Add the agent **Voice tab** (`AgentVoiceTab`) -- how the agent sounds and
listens. **Scope: per agent**; every user of the agent gets the same voice
and call settings. The tab opens with an info box, then three sub-tabs that
are always usable: **Voice** (the voice that reads chat replies), **Calls**
(how live calls behave), and **Dictation** (the speech-to-text side). Calls
and Dictation each open with a box of two switches, **In-App Chat** and
**Embed Widget**, that show or hide the chat's call button or microphone
button on that surface. Settings > Capabilities no longer has these switches.
The two In-App Chat switches are only here; the Embed tab (`AgentEmbedTab`)
still has its own switches for the two Embed Widget fields. **Save** is
pinned at the bottom on every sub-tab. This is one tab in the agent-settings
family indexed by `/iblai-vibe-agent`; it shares the `AgentSettingsProvider`
wrapper with every other tab.

**Voice** -- no switch here: the voice source (**Browser**, **Google**,
**ibl.ai**, **OpenAI**), the voice, and for OpenAI and Google the **Voice
Instructions** card (here filled from the "Warm and Encouraging" example).

![Voice tab -- Voice sub-tab](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-voice/iblai-vibe-agent-voice-1-voice.png)

**Voice picker** -- the voice trigger opens a searchable list of the
provider's voices, each with a play button for a sample.

![Voice tab -- voice picker](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-voice/iblai-vibe-agent-voice-2-voice-picker.png)

**Calls** -- the **Voice Calls** box (In-App Chat and Embed Widget
switches), then call style (Live Conversation / Step-by-Step), spoken
language, AI provider, **Smart Document Retrieval**, and the voice used on
calls, with **Reset** and **Save**. Smart Document Retrieval saves the call
configuration's `use_function_calling_for_rag`: on, a call that runs step by
step looks things up in the agent's documents only when it needs to (live
conversation calls always work this way). It moved here from Settings >
Capabilities and is saved by this sub-tab's **Save**.

![Voice tab -- Calls sub-tab: the Voice Calls box with its In-App Chat and Embed Widget switches, then the call settings](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-voice/iblai-vibe-agent-voice-3-voice-call.png)

**Dictation** -- the **Dictation** box (In-App Chat and Embed Widget
switches for the chat-box microphone), then **Spoken Language** for
transcription (unset = Detect Automatically) and **Transcription Hints**,
with **Save**.

![Voice tab -- Dictation sub-tab: the Dictation box with its two switches, then the spoken language and hints](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-voice/iblai-vibe-agent-voice-4-dictation.png)

The settings under each box grey out only while both of its switches are
off. A field the agent's settings leave out reads as on.

> **Common setup (brand, conventions, env files, verification):** see [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/docs/skill-setup.md).

## Prerequisites

- Auth set up (`/iblai-vibe-auth`, or vibe-starter).
- `AgentSettingsProvider` wraps the route (`/iblai-vibe-agent` §1).
- `@iblai/iblai-js@^3.1.0` (resolves `@iblai/web-containers` 2.1.0, which
  has the per-surface switches, the ibl.ai source, and the Calls and
  Dictation sub-tabs). The four switches work on any platform version.
- A real agent UUID. Ask the user; never invent one.

## Step 1: Check Environment

Look for `iblai.env` in the project root with `PLATFORM`, `DOMAIN`, and
`TOKEN`. If it is missing, tell the user:
"You need an `iblai.env` with your platform configuration. Download the
template and fill in your values:
`curl -o iblai.env https://raw.githubusercontent.com/iblai/base/refs/heads/main/iblai.env`"

## Step 2: Copy the provider logos

The Google, ibl.ai, and OpenAI cards load their logos from your app's
`public/` root, and the SDK ships them in its own `public/`. Copy them once
(pnpm layout shown; with npm the folder is
`node_modules/@iblai/web-containers/public`):

```bash
cp node_modules/.pnpm/@iblai+web-containers@*/node_modules/@iblai/web-containers/public/llm-* public/
```

Expect `public/llm-google-provider.svg`, `public/llm-iblai-provider.png`, and
`public/llm-openai-provider-2.svg`. Without them the cards show broken images.

## Step 3: Mount `AgentVoiceTab`

```tsx
// app/(app)/agents/[mentorId]/voice/page.tsx
"use client";

import { AgentVoiceTab } from "@iblai/iblai-js/web-containers/next";

export default function AgentVoicePage() {
  return (
    <div className="flex h-full flex-col bg-white">
      <AgentVoiceTab />
    </div>
  );
}
```

The tab reads `tenantKey`, `mentorId`, `username`, and `enableRBAC` from
`AgentSettingsProvider` and does its own fetching and saving. No props are
required. To land on another sub-tab (for example from a URL), pass
`defaultSubTab`: `'voice'` (Voice), `'callConfig'` (Calls) or `'settings'`
(Dictation). The values kept their old names.

```tsx
import { AgentVoiceTab } from "@iblai/iblai-js/web-containers/next";

<AgentVoiceTab defaultSubTab="callConfig" />;
```

## Step 4: Customize Labels (Optional)

The tab renders with the default agent-facing copy (`AGENT_VOICE_TAB_LABELS`,
localized through the SDK's i18n). Pass a partial `labels` object to change
any string:

```tsx
import { AgentVoiceTab } from "@iblai/iblai-js/web-containers/next";

<AgentVoiceTab
  labels={{
    header: { description: "Pick the voice your agent speaks with." },
    subTabs: { callConfig: "Phone Calls" },
  }}
/>;
```

Label groups (`VoiceTabLabels`):

| Group | Covers |
|---|---|
| `header` | `title`, `description` |
| `infoBox` | the box that opens the tab |
| `surfaces` | `inAppChat`, `embedWidget` (`label`, `tooltip` each): the column labels over the switches |
| `switchName` | `(name, surface) => string`: each switch's accessible name, such as "Voice Calls, In-App Chat" |
| `capability` | the Voice Calls box: `title`, `description`, `offHint` |
| `subTabs` | `voice`, `callConfig`, `settings` (Voice, Calls, Dictation) |
| `mentorVoice` | Voice sub-tab: `description`, `providerLabel`, `providerTooltip`, `providerOptions.{browser,google,iblai,openai}`, `voicePickerLabel`, picker strings (`searchPlaceholder`, `emptyState`, `errorState`, `previewAria`), `saveButton`, `savingButton`, `instructions` (`label`, `tooltip`, `placeholder`, `helpText`, `presetsLabel`, `presets.{warm,calm,energetic}`) |
| `callConfig` | Calls sub-tab: `description`, `info`, `fields.{mode,language,llmProvider,ttsProvider,sttProvider,useFunctionCalling,enableVideo}` (`useFunctionCalling` names the Smart Document Retrieval switch), `resetButton`, `saveCreate`, `saveUpdate`, `savingButton` |
| `transcription` | Dictation sub-tab: `toggle` (the Dictation box: `title`, `description`, `offHint`), `language`, `instructions`, `saveButton`, `savingButton` |
| `toasts` | `voiceSaved`, `voiceError`, `callConfigSaved`, `callConfigError`, `transcriptionSaved`, `transcriptionError`, `turnedOn(name, surface)`, `turnedOff(name, surface)` |

`@iblai/web-containers` 2.0.0 removed `mentorVoice.title`, `callConfig.title`
and `transcription.toggle.label` / `tooltip` / `helpText`. A host that still
passes them in `labels` should delete them: the tab no longer reads them, and
TypeScript rejects them as unknown keys in an object literal.

## Step 5: Use MCP Tools for Customization

```
get_component_info("AgentVoiceTab")
get_component_info("AgentSettingsProvider")
```

## `<AgentVoiceTab>` Props

Import from `@iblai/iblai-js/web-containers/next`.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `labels` | `DeepPartial<VoiceTabLabels>` | No | Override user-visible strings |
| `defaultSubTab` | `'voice' \| 'callConfig' \| 'settings'` | No | Sub-tab to open on (Voice, Calls, Dictation). Defaults to `'voice'` |
| `renderPromptContent` | `(content: string) => ReactNode` | No | Renders the Voice Instructions card (e.g. as Markdown); plain text otherwise |
| `tenantKey` | `string` | No | Identity override. Falls back to `AgentSettingsProvider` |
| `mentorId` | `string` | No | Identity override (agent UUID). Falls back to `AgentSettingsProvider` |
| `username` | `string` | No | Identity override. Falls back to `AgentSettingsProvider` |
| `enableRBAC` | `boolean` | No | Field-level permission gating; with it on, a switch whose field has `write: false` is locked. Falls back to `AgentSettingsProvider` |

## Related Exports

From `@iblai/iblai-js/web-containers/next`:

- `AGENT_VOICE_TAB_LABELS` -- the default label bundle.
- `VoiceTabLabels`, `AgentVoiceTabProps` -- types.

## How it saves

| Control | Saves | When |
|---|---|---|
| **Voice Calls** switches (Calls) | `show_voice_call` (In-App Chat) or `embed_show_voice_call` (Embed Widget) on the agent's settings | On the click, only its own field |
| **Dictation** switches (Dictation) | `show_voice_record` or `embed_show_voice_record` | On the click, only its own field |
| **Save** (Voice) | `voice_provider`, the picked `*_voice`, `voice_instructions` | On click; a voice is sent only when picked this session, instructions only when changed |
| **Save** (Calls) | The agent's call configuration (`mode`, `language`, `llm_provider`, call voice, `use_function_calling_for_rag`) | On click; creates the row if missing; needs an org admin |
| **Save** (Dictation) | `transcription_language`, `transcription_instructions` | On click; blocked over 1000 characters |

Clearing instructions or hints sends `""`; an absent field means "no change".

A switch toasts "Voice Calls turned off for Embed Widget" and the like. A
refused save turns it back and shows one sentence under the box ("Couldn't
update Voice Calls. Try again."), no toast. With `enableRBAC` on, a switch
whose field has `permissions.field[field].write === false` is locked, with
"You don't have permission to change this." under the box.

The platform writes the whole agent on every settings save, so the tab sends
its settings saves one at a time: two in flight together could undo each
other. A click made during a save shows at once and is sent when that save
has finished; a waiting Voice or Dictation **Save** reads "Saving…". A switch
save never resets unsaved Voice or Dictation edits.

## Platform data

| Hook | Purpose |
|---|---|
| `useGetMentorSettingsQuery` | Voice fields plus the embedded `call_configuration` |
| `useEditMentorMutation` | Voice and Dictation saves and the four switches, on the agent's settings |
| `useCreateCallConfigurationMutation`, `useUpdateCallConfigurationMutation` | Calls sub-tab, Smart Document Retrieval included |

REST twin: [`/iblai-api-agent-voice`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-api-agent-voice/SKILL.md)
(settings voice fields, call configurations, and the voice catalog).

## Step 6: Verify

Run `/iblai-vibe-ops-test` before telling the user the work is ready, and give
every page, control and state this skill added its own checkpoint — one Playwright
test titled `<ID> · …` in `e2e/journeys/`, listed in `e2e/coverage.json`;
`pnpm e2e:coverage` must pass:

1. `pnpm build` -- must pass with zero errors.
2. `pnpm test` -- vitest must pass.
3. Touch test `/agents/<uuid>/voice` in the browser as `/iblai-vibe-ops-test` says: the
   info box and the Voice, Calls and Dictation sub-tabs render; the four source cards
   show their logos and the voice picker lists voices with working previews; on Calls,
   turn Voice Calls off for Embed Widget ("Voice Calls turned off for Embed Widget") and
   back on; read the console; check the phone width.

## Important Notes

- **Two capabilities, two surfaces each**: Voice Calls (`show_voice_call`,
  `embed_show_voice_call`) is the user talking with the agent on a call;
  Dictation (`show_voice_record`, `embed_show_voice_record`) is the user
  speaking into the chat box. Each field only shows or hides a button in that
  surface's chat. Settings > Capabilities no longer has these rows, and its
  **Save** no longer sends them. `AgentEmbedTab` keeps its own embed voice
  switches for hosts that render it.
- **Voice sources**: **Browser** speaks on the listener's device (no voice to
  pick). **ibl.ai** falls back to the organization's default voice.
  **OpenAI** and **Google** need a voice and accept **Voice Instructions**.
- **Voice instructions cost more on OpenAI**: any instructions switch OpenAI
  to an instruction-capable speech model; the help text says so. The
  1000-character cap is a UI guardrail.
- **Screen sharing** lives on its own tab (`/iblai-vibe-agent-screenshare`)
  and always uses Live Conversation.
- **Shared provider**: mount `AgentSettingsProvider` once at the layout level
  (`/iblai-vibe-agent` §1); do not wrap each tab.
- **Peer deps**: `sonner` and `@iblai/iblai-web-mentor`
  (`pnpm add sonner @iblai/iblai-web-mentor`).
- **Brand guidelines**: [BRAND.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/BRAND.md)
