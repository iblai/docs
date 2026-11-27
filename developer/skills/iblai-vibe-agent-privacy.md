# /iblai-vibe-agent-privacy

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-vibe-agent-privacy/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-vibe-agent-privacy/SKILL.md). This page is generated — edit it in the repository, not here.

Add the agent **Privacy tab** -- "Control what personal information the
agent sees and how Incognito works for its conversations." **Scope: per
agent.** Every control saves at once to the agent's settings, so it applies
to everyone who chats with the agent. The tab opens with a gray info box
(`privacy-info-box`): "Decide how private conversations with this agent
are. PII Filtering looks for personal details, like names, emails and phone
numbers. PHI Filtering looks for health details, like conditions and
medications. Incognito keeps chats out of history." Below it, two sub-tabs
behind one pill switcher:

- **Privacy Filtering** — two cards, **PII Filtering** and **PHI
  Filtering**, each with its own switch and a status line (below). Open a
  card for what to do when it finds something (**Allow**, **Redact**,
  **Mask**, **Block**), its block message and its **Entity Types** chips
  (for PII: Person, Email, Phone, SSN, Credit Card, Location, Date / Time,
  Passport, Driver's License, IP Address, IBAN, Medical License, Bank
  Number). Under the cards, **Also Filter AI Responses**.
- **Incognito** — one choice over how Incognito works for this agent:
  **Users Decide**, **Always Incognito**, or **Never Incognito**. This
  replaces the old Private Mode switch that lived on the Settings tab.

This is one tab in the agent-settings family indexed by `/iblai-vibe-agent`;
it shares the `AgentSettingsProvider` wrapper with every other tab (optional
for this tab — identity can be passed as props).

**Filtering off** -- each card's switch turns its kind of filtering on
(the first one on also turns the Privacy Router on). The status line is a
full sentence with no "On." or "Off." in front, since the switch shows
that. Off, it says what reaches the AI: "Names, emails, phone numbers and
other personal details reach the AI just as people type them." An opened
card that is off shows only a hint: "Turn it on to look for personal
details in messages and choose what happens to them."

![Privacy tab -- off](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-privacy/iblai-vibe-agent-privacy-1-off.png)

**On, with Redact** -- the status line says what happens to which
details: "Names, emails, phone numbers and other personal details are
replaced with a label before the AI sees them.", or "Only the personal
details you picked (3 kinds) are …" when entity types are picked. **When
PII Is Detected** picks the action (the line under it explains it),
**Entity Types** picks what to detect (none selected means the defaults),
and **Also Filter AI Responses** checks the agent's replies too.

![Privacy — Privacy Filtering sub-tab, PII Filtering on](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-privacy/iblai-vibe-agent-privacy.png)

**Actions** -- **Allow** (detect and log, leave the message unchanged),
**Redact** (replace with the type, e.g. `[EMAIL_ADDRESS]`), **Mask** (replace
with asterisks), **Block** (reject the message).

![Privacy tab -- actions](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-privacy/iblai-vibe-agent-privacy-3-actions.png)

**Block** -- adds **PII Block Message**, the text the user sees instead
of an answer. It saves when the field loses focus.

![Privacy tab -- Block](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-privacy/iblai-vibe-agent-privacy-4-block.png)

**Incognito** -- the three cards, offered only when the organization allows
it (Prerequisites).

![Privacy — Incognito sub-tab](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-privacy/iblai-vibe-agent-privacy-incognito.png)

> **Common setup (brand, conventions, env files, verification):** see [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/docs/skill-setup.md).

## Related Incognito surfaces

Incognito is one feature with four tiers; this tab is the **agent**
tier. `/iblai-vibe-incognito` explains how they resolve and mounts the
in-chat toggle.

- **`/iblai-vibe-incognito`** — the chat-header **Incognito** toggle and
  its "Enable Incognito for this chat?" dialog, plus the tier order.
- **`/iblai-vibe-profile`** — the user's own **Privacy** tab: their
  default (Normal / Anonymized / Incognito).
- **`/iblai-vibe-account`** — organization settings → **Advanced** →
  "Allow users to control chat privacy", the gate that makes the
  Incognito sub-tab (and the toggle) exist at all.

## Prerequisites

- Auth set up (`/iblai-vibe-auth`, or vibe-starter).
- `AgentSettingsProvider` wraps the route (`/iblai-vibe-agent` §1), **or** pass
  `tenantKey` / `mentorId` / `username` as props.
- `@iblai/iblai-js@^3.1.0` (resolves `@iblai/web-containers` 2.1.0). Check with
  `pnpm why @iblai/web-containers`.
- A real agent UUID. Ask the user; never invent one.
- The **Incognito** sub-tab appears only when the organization allows
  user chat-privacy control (`allow_user_chat_privacy_control`). With
  the gate off there is no Incognito to govern: below the info box the
  tab shows the **Privacy Filtering** content (both cards and **Also
  Filter AI Responses**) on its own, with no sub-tab switcher.

## Step 1: Check Environment

Look for `iblai.env` in the project root with `PLATFORM`, `DOMAIN`, and
`TOKEN`. If it is missing, tell the user:
"You need an `iblai.env` with your platform configuration. Download the
template and fill in your values:
`curl -o iblai.env https://raw.githubusercontent.com/iblai/base/refs/heads/main/iblai.env`"

## Step 2: Mount `AgentPrivacyTab`

```tsx
// app/(app)/agents/[mentorId]/privacy/page.tsx
"use client";

import { AgentPrivacyTab } from "@iblai/iblai-js/web-containers/next";

export default function AgentPrivacyPage() {
  return (
    <div className="flex h-full flex-col bg-white">
      <AgentPrivacyTab />
    </div>
  );
}
```

The tab reads `tenantKey`, `mentorId`, `username`, `enableRBAC` and
`executeGatedAction` from `AgentSettingsProvider`, loads the agent settings
and the organization's chat-privacy gate, and decides which sub-tabs to offer
(see **Which sub-tabs appear** below). Each can be passed as a prop instead,
for example to mount the tab outside the provider:

```tsx
import { AgentPrivacyTab } from "@iblai/iblai-js/web-containers/next";

export function PrivacyPanel(props: { org: string; agentId: string; username: string }) {
  return (
    <AgentPrivacyTab tenantKey={props.org} mentorId={props.agentId} username={props.username} />
  );
}
```

With `enableRBAC` on, each control follows the agent's field permissions
(`enable_privacy_router`, `filter_pii`, `privacy_action`,
`privacy_response`, `privacy_entities`, `filter_phi`, `phi_action`,
`phi_response`, `phi_entities`, `enable_privacy_output_filter`,
`disable_chathistory`, `disable_privacy_mode`) and is disabled when the user
may not edit it.
`executeGatedAction` wraps every save, so the host can put a paywall or
confirmation in front of it.

## Step 3: Customize Labels (Optional)

The tab renders with the default agent-facing copy
(`AGENT_PRIVACY_TAB_LABELS`, localized through the SDK's i18n). Pass a
partial `labels` object to change any string:

```tsx
import { AgentPrivacyTab } from "@iblai/iblai-js/web-containers/next";

<AgentPrivacyTab
  labels={{
    header: { title: "Data privacy", description: "Filter PII; decide how Incognito works." },
    subTabs: { filtering: "Personal data", incognito: "Incognito" },
    fields: { action: { options: { allow: "Log only" } } },
    incognito: {
      options: { always: { title: "Always private", description: "Nothing is ever saved." } },
    },
  }}
/>;
```

Label groups (`PrivacyTabLabels`): `header` (`title`, `description`),
`infoBox` (optional; `infoBox: ""` hides the box), `categories` (`pii` and
`phi`, each `title`, `expand`, `offHint`, `summary`, whose `all` and
`selected` receive the action's key: `redact`, `mask`, `block` or `allow`),
`switchState` (`on`, `off`), `groups` (`action`, `entityTypes`,
`aiResponses`, `privacyLog`), `fields` -- `action` (`label`, `tooltip`,
`options` and `descriptions` per action), `blockMessage` (`label`,
`placeholder`), `entities` (`tooltip`, `emptyHint`, `labels` per entity
type), `outputFilter` (`label`, `description`) -- `phiFields` (`action`,
`blockMessage`, `entities`), `privacyLog` (`description`), `subTabs`
(`filtering`, `incognito`), `incognito` (`description`, `groupAriaLabel`,
`options` per card), `noAccess`, and `toasts` (`updateSuccess`,
`updateError`).

## Step 4: Use MCP Tools for Customization

```
get_component_info("AgentPrivacyTab")
get_component_info("AgentSettingsProvider")
get_component_info("ChatPrivacyToggle")
```

## `<AgentPrivacyTab>` Props

Import from `@iblai/iblai-js/web-containers/next`.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `labels` | `DeepPartial<PrivacyTabLabels>` | No | Override user-visible strings |
| `tenantKey` | `string` | No | Organization key; defaults to `AgentSettingsProvider` |
| `mentorId` | `string` | No | Agent UUID; defaults to `AgentSettingsProvider` |
| `username` | `string` | No | Signed-in username; defaults to `AgentSettingsProvider` |
| `enableRBAC` | `boolean` | No | Follow the agent's field permissions; defaults to the provider, else `false` |
| `executeGatedAction` | `(fn: () => unknown) => unknown` | No | Wrap every save (paywall, confirmation); defaults to the provider |
| `rbacPermissions` | `object` | No | Gates **View Privacy Log** (`/mentors/{mentor_id}/#view_privacy_flags`); defaults to the provider |

## What each sub-tab renders

### Privacy Filtering

- **PII Filtering** and **PHI Filtering** cards (`privacy-pii-card`,
  `privacy-phi-card`). Each header has a chevron that opens the card, the
  title, a status line and a switch (`privacy-filter-pii-toggle`,
  `privacy-filter-phi-toggle`). Turning one on while both are off turns the
  Privacy Router on for that kind only; turning the last one off turns the
  router off. An open card that is off shows only its hint.
- **The status line** (`privacy-pii-summary`, `privacy-phi-summary`) is a
  full sentence. For PII: off, "Names, emails, phone numbers and other
  personal details reach the AI just as people type them."; on, "Names,
  emails, phone numbers and other personal details are …", or "Only the
  personal details you picked (2 kinds) are …" when entity types are picked.
  The sentence ends with what the action does: Redact "replaced with a label
  before the AI sees them", Mask "hidden behind asterisks", Block "caught,
  and the message is stopped before it reaches the AI", Allow "noticed, but
  still reach the AI as written". The PHI card says the same of
  "Conditions, medications, medical record numbers and other health
  details".
- **When PII Is Detected** — a select with an inline description of the
  chosen action:

  | Action | Wire value | What happens |
  |---|---|---|
  | Allow | `allow` | Detected but left untouched — the message passes through unchanged. For monitoring or auditing without altering messages |
  | Redact (default) | `redact` | Replaced with its type — e.g. `Email [EMAIL_ADDRESS]` |
  | Mask | `mask` | Hidden behind asterisks — e.g. `Email ***********` |
  | Block | `block` | The message is rejected and your block message is shown |

- **PII Block Message** — a textarea shown only for **Block**; saves on
  blur.
- **Entity Types** — the 13 chips, multi-select. With none selected the
  hint reads *"Using defaults."*
- The PHI card has the same three controls: **When PHI Is Detected**
  (default **Allow**), **PHI Block Message**, and 12 chips (Condition,
  Medication, Treatment, …); with none selected it looks for all 12.
- **Also Filter AI Responses** — "Check the agent's replies for PII and PHI
  too. May slightly slow down replies."
- **View Privacy Log** — opens the log of what was detected, for a user
  whose `rbacPermissions` hold `/mentors/{mentor_id}/#view_privacy_flags`.

### Incognito

*"Incognito keeps a conversation out of chat history and memory.
Choose how it works for this agent."* Three radio cards:

| Card | Flags saved | Effect in the chat header |
|---|---|---|
| **Users Decide** | `disable_chathistory: false`, `disable_privacy_mode: false` | Users can turn Incognito on for any conversation; otherwise chats are saved as usual |
| **Always Incognito** | `disable_chathistory: true` | Every conversation is incognito, for anyone. The Incognito pill shows **locked on** |
| **Never Incognito** | `disable_privacy_mode: true` | Incognito isn't available for this agent; the toggle disappears. Users whose profile default is Incognito are saved **anonymously** instead |

One control writes both flags in the same PUT, so the contradictory
"always *and* never" pair cannot be configured from here. A record that
still carries both (older data) **behaves as Always Incognito**: the
backend checks `disable_chathistory` first, answers `mode: disabled`,
`source: mentor`, and writes no history — even though the tab displays
it as Never Incognito. Choosing any card rewrites both flags and clears
the mismatch.

## Which sub-tabs appear

| Condition | Result |
|---|---|
| Organization gate `allow_user_chat_privacy_control` off | Incognito sub-tab not offered |
| `enableRBAC` and **no** filtering field (PII, PHI or output filter) is readable (`permissions.field`) | Privacy Filtering not offered |
| `enableRBAC` and **either** Incognito flag is unreadable | Incognito not offered |
| Only one sub-tab offered | It renders on its own, with no switcher (no `privacy-sub-tab-*` triggers) |
| Nothing readable | The tab shows a single notice: *"You don't have permission to view this agent's privacy settings."* |

Members typically see Incognito read-only and no Privacy Filtering. Writes
stay per control: a field the user may read but not write renders
disabled.

## Settings fields

Read with `GET` and written with `PUT` on
`${dmUrl}/api/ai-mentor/orgs/{org}/users/{username}/mentors/{mentor_unique_id}/settings/`
(a partial update — send only what changes):

| Field | Type | Sub-tab |
|---|---|---|
| `enable_privacy_router` | `boolean` | Privacy Filtering — on while either card is on |
| `filter_pii`, `filter_phi` | `boolean` | Privacy Filtering — each card's switch (with the router) |
| `privacy_action`, `phi_action` | `"allow" \| "redact" \| "mask" \| "block"` | Privacy Filtering |
| `privacy_response`, `phi_response` | `string` | Privacy Filtering — the block messages |
| `privacy_entities`, `phi_entities` | `string[]` (`PERSON`, `EMAIL_ADDRESS`, …; `MEDICATION`, …) — `[]` means defaults (all 12 for PHI) | Privacy Filtering |
| `enable_privacy_output_filter` | `boolean` | Privacy Filtering |
| `disable_chathistory` | `boolean` | Incognito — Always |
| `disable_privacy_mode` | `boolean \| null` (`null` = off on older records) | Incognito — Never |

## Related Exports

From `@iblai/iblai-js/web-containers/next`:

- `AGENT_PRIVACY_TAB_LABELS` -- the default label bundle.
- `AgentPrivacyTabProps`, `PrivacyTabLabels` -- types.

From `@iblai/iblai-js/web-containers`: `ChatPrivacyToggle` -- the chat-header
toggle that reacts to the Incognito policy (`/iblai-vibe-incognito`).

From `@iblai/iblai-js/data-layer`:

- `PRIVACY_ACTIONS` (`allow`, `redact`, `mask`, `block`),
  `PRIVACY_ENTITY_TYPES` (`PERSON`, `EMAIL_ADDRESS`, `PHONE_NUMBER`, `US_SSN`,
  `CREDIT_CARD`, `LOCATION`, `DATE_TIME`, `US_PASSPORT`, `US_DRIVER_LICENSE`,
  `IP_ADDRESS`, `IBAN_CODE`, `MEDICAL_LICENSE`, `US_BANK_NUMBER`),
  `DEFAULT_PRIVACY_ACTION`, `DEFAULT_PRIVACY_RESPONSE`, and the
  `PrivacyAction` / `PrivacyEntityType` types.
- `PHI_ENTITY_TYPES` (`DISEASE_DISORDER`, `MEDICATION`,
  `THERAPEUTIC_PROCEDURE`, `BIOLOGICAL_STRUCTURE`, `CLINICAL_EVENT`,
  `BIOLOGICAL_ATTRIBUTE`, `HISTORY`, `FAMILY_HISTORY`, `MEDICAL_RECORD_NUMBER`,
  `HEALTH_PLAN_BENEFICIARY_NUMBER`, `ACCOUNT_NUMBER`, `DEVICE_SERIAL_NUMBER`),
  `DEFAULT_PHI_ACTION` (`allow`), and the `PhiEntityType` type -- the PHI
  card's counterparts.
- `isPrivacyBlockedError`, `PRIVACY_BLOCKED_ERROR_KIND` -- recognise the
  `privacy_blocked` error a **Block** action sends over the chat stream, to
  style it apart from a moderation block.
- `MentorChatPrivacyPublicSettings` -- `disable_privacy_mode` as members read
  it from the agent's public settings.

## Platform data

| Hook | Purpose |
|---|---|
| `useGetMentorSettingsQuery` | Current privacy fields and field permissions |
| `useEditMentorJsonMutation` | Save a change (JSON, so `privacy_entities` stays an array) |
| `useGetTenantChatPrivacyConfigQuery` | The organization gate the Incognito sub-tab depends on |

REST twin: [`/iblai-api-agent-privacy`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-api-agent-privacy/SKILL.md)
(also the `privacy-flags` audit log of every detection, which **View
Privacy Log** opens).

## Step 5: Verify

Run `/iblai-vibe-ops-test` before telling the user the work is ready, and give
every page, control and state this skill added its own checkpoint — one Playwright
test titled `<ID> · …` in `e2e/journeys/`, listed in `e2e/coverage.json`;
`pnpm e2e:coverage` must pass:

1. `pnpm build` -- must pass with zero errors.
2. `pnpm test` -- vitest must pass.
3. Touch test `/agents/<uuid>/privacy` in the browser as `/iblai-vibe-ops-test` says,
   signed in as an org admin: the info box shows; turn PII Filtering on and read
   its status line; pick **Block** -- the PII Block Message field appears; pick
   each Incognito card; read the console; check the phone width.

Stable selectors: `privacy-info-box`, `privacy-sub-tab-filtering`,
`privacy-sub-tab-incognito`, `privacy-filter-{pii|phi}-toggle`,
`privacy-{pii|phi}-summary`, `privacy-{pii|phi}-expand`,
`privacy-output-filter-toggle`, `privacy-incognito-{users|always|never}`
(with `aria-pressed`), `privacy-no-access`.

## Important Notes

- **Saves on every click**: there is no Save button. Try settings on a test
  agent.
- **Allow still detects**: detections are logged to the `privacy-flags`
  audit log; only the message is left alone.
- **JSON settings endpoint**: all fields are persisted via the JSON
  `/settings/` mutation (`useEditMentorJsonMutation`), not the
  multipart variant, so `privacy_entities` round-trips cleanly. The tab
  refetches agent settings after each save because that mutation does
  not invalidate the settings cache tag.
- **Incognito saves refresh the chat header**: choosing a card also
  invalidates `ChatPrivacyEffective` and the agent's
  `mentorPublicSettings` caches, so an open chat's Incognito pill locks,
  unlocks or disappears without a reload. Custom UI writing the two
  flags must do the same.
- **Private Mode moved**: the switch that used to sit on the Settings
  tab's capabilities is gone; the Incognito sub-tab here is its
  replacement, and the user-facing name is now **Incognito** everywhere.
- **Shared provider**: mount `AgentSettingsProvider` once at the layout
  level (`/iblai-vibe-agent` §1); do not wrap each tab.
- **Peer deps**: `sonner` and `@iblai/iblai-web-mentor`
  (`pnpm add sonner @iblai/iblai-web-mentor`).
- **Brand guidelines**: [BRAND.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/BRAND.md)
