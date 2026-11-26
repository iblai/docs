# /iblai-vibe-agent-support

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-vibe-agent-support/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-vibe-agent-support/SKILL.md). This page is generated — edit it in the repository, not here.

Add the agent **Support tab** -- a human-support inbox for one agent. When
a user asks the agent for a person, the agent files a ticket; admins
review, reply to and resolve it here. **Scope: per agent.** The **Human
Support** switch turns the hand-over on for this agent (it saves the agent's
`capability_human_support` flag), and the inbox lists only this agent's
tickets. This is one tab in the agent-settings family indexed by
`/iblai-vibe-agent`; it shares the `AgentSettingsProvider` wrapper with every
other tab.

**Tickets** -- the info box with the **Human Support** switch, a user filter
(search the organization's users), a status filter (All / Open / In
Progress / Closed), and the inbox: requester,
subject, when, a preview, and a status icon. The icon is grey; hover it (or
use a screen reader) for the word. The inbox shows as many tickets as fit the
space and pages the rest, so it never scrolls (between 768 and 1023px wide it
shows 10 a page instead). With no tickets the tab shows
one dashed empty box.

![Support tab -- tickets](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-support/iblai-vibe-agent-support-1-tickets.png)

**A ticket** -- reads like an email: the subject, requester and **Status** at
the top, then the thread (the request the agent wrote first, then every
reply, oldest first; HTML and Markdown render sanitized), and a reply box at
the bottom with a **To** line. Replies from anyone but the requester show as
**Support Team**.

![Support tab -- a ticket](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-support/iblai-vibe-agent-support-2-ticket.png)

**Reply** -- type in the reply box and press **Send Reply** (bottom right);
the message joins the thread.

![Support tab -- reply](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-support/iblai-vibe-agent-support-3-reply.png)

**Status** -- Open, In Progress or Closed. A closed ticket replaces the
reply box with "This ticket is closed. Set it back to Open or In Progress to
reply."

![Support tab -- a closed ticket](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-support/iblai-vibe-agent-support-4-status.png)

**On a phone** (below 768px) -- each ticket is a card with its Status and
From, the switch sits on its own row named **Human Support**, the filters
open as sheets from the bottom, and a "Page x of y" pager replaces the page
numbers.
Tapping a ticket opens it full screen, with Back.

![Support tab on a phone](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-support/iblai-vibe-agent-support-5-phone.png)
![A ticket on a phone](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-support/iblai-vibe-agent-support-6-phone-ticket.png)

> **Common setup (brand, conventions, env files, verification):** see [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/docs/skill-setup.md).

## Prerequisites

- Auth set up (`/iblai-vibe-auth`, or vibe-starter).
- `AgentSettingsProvider` wraps the route (`/iblai-vibe-agent` §1). The tab
  throws without it: it has no identity props.
- `@iblai/iblai-js@^3.1.0` (resolves `@iblai/web-containers` 2.1.0). Check with
  `pnpm why @iblai/web-containers`.
- A real agent UUID. Ask the user; never invent one.
- A platform that returns `capability_human_support` on the agent's
  settings (API version 4.467.0 or later), and a person who may read it.
  Otherwise the switch is hidden; the inbox still works.

## Step 1: Check Environment

Look for `iblai.env` in the project root with `PLATFORM`, `DOMAIN`, and
`TOKEN`. If it is missing, tell the user:
"You need an `iblai.env` with your platform configuration. Download the
template and fill in your values:
`curl -o iblai.env https://raw.githubusercontent.com/iblai/base/refs/heads/main/iblai.env`"

## Step 2: Mount `AgentHumanSupportTab`

```tsx
// app/(app)/agents/[mentorId]/support/page.tsx
"use client";

import { AgentHumanSupportTab } from "@iblai/iblai-js/web-containers/next";

export default function AgentSupportPage() {
  return (
    <div className="flex h-full flex-col bg-white">
      <AgentHumanSupportTab />
    </div>
  );
}
```

The tab reads `tenantKey`, `mentorId`, and `username` from
`AgentSettingsProvider` and does its own fetching and saving. No props are
required. To use your own pagination under the inbox, pass
`PaginationComponent` (it gets `currentPage`, `totalPages`, `onPageChange`,
`disabled`, `disableNumberedButtons`; the default is the SDK's
`IblPagination`). Phones always get the SDK's "Page x of y" pager.

The tab measures its wrapper to decide how many rows fit, so give it a set
height (here `h-full` inside a full-height layout, as in `/iblai-vibe-agent`
§1). On phones the tab's explanation moves behind an info button in your
header when the header has an empty `<div data-activity-info-slot />`;
without one it stays above the tab.

## Step 3: Customize Labels (Optional)

The tab renders with the default agent-facing copy
(`AGENT_HUMAN_SUPPORT_TAB_LABELS`, localized through the SDK's i18n). Pass a
partial `labels` object to change any string:

```tsx
import { AgentHumanSupportTab } from "@iblai/iblai-js/web-containers/next";

<AgentHumanSupportTab
  labels={{
    header: { title: "Help desk" },
    detail: { sendReply: "Reply" },
  }}
/>;
```

Label groups (`HumanSupportTabLabels`): `header` (`title`, `description`,
`infoBox`), `filters` (`searchUser`, `allUsers`, `allStatuses`), `status` (`open`,
`inProgress`, `closed`), `list` (`fromLabel`, `noTickets`, `untitled`),
`detail` (`ticketTitle`, `selectPrompt`, `statusLabel`, `requestSection`,
`conversationSection`, `supportTeam`, `noMessages`, `toLabel`, `toRecipient`,
`replyPlaceholder`, `sendReply`, `sending`, `closedNotice`), and `toasts`
(`replySent`, `replyError`, `statusUpdated`, `statusError`).

The `toggle` group was removed in `@iblai/web-containers` 2.0.0; a host that
passes it removes it. The switch is named after its capability, "Human
Support", and confirms a save with "Human Support turned on" / "Human Support
turned off", like every other capability switch. Those strings come from the
SDK's own catalogue, not from `labels`.

## Step 4: Use MCP Tools for Customization

```
get_component_info("AgentHumanSupportTab")
get_component_info("AgentSettingsProvider")
```

## `<AgentHumanSupportTab>` Props

Import from `@iblai/iblai-js/web-containers/next`.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `labels` | `DeepPartial<HumanSupportTabLabels>` | No | Override user-visible strings |
| `PaginationComponent` | `ComponentType<{ currentPage, totalPages, onPageChange, disabled, disableNumberedButtons }>` | No | Pagination under the inbox. Defaults to the SDK's `IblPagination` |

## Related Exports

From `@iblai/iblai-js/web-containers/next`:

- `AGENT_HUMAN_SUPPORT_TAB_LABELS` -- the default label bundle.
- `AgentHumanSupportTabProps`, `HumanSupportTabLabels`,
  `SupportTicketsFilter` (the status and requester filter state) -- types.

## How it saves

Paths are under `/api/ai-mentor/orgs/{org}/users/{username}/`.

| Action | Request |
|---|---|
| Load | `GET support-tickets/?mentor_id=<uuid>&page=&page_size=<rows that fit>` (plus `status` and `username` from the filters) |
| Open a ticket | `GET support-ticket-messages/?ticket={id}` |
| **Send Reply** | `POST support-ticket-messages/` with the ticket and the message |
| Status Open / In Progress | `PATCH support-tickets/{id}/` with `status` |
| Status Closed | `POST support-tickets/{id}/close/` |
| **Human Support** switch | `PUT mentors/{uuid}/settings/` with `capability_human_support` only (plus `can_use_tools: true` when turning it on for an agent that could not use tools yet; `false` is never sent) |

There is no create here: the agent files tickets during chats. The tab has
no delete either; the REST twin has one.

## Platform data

| Hook | Purpose |
|---|---|
| `useGetSupportTicketsQuery` | The inbox (paged) |
| `useGetSupportTicketMessagesQuery` / `useCreateSupportTicketMessageMutation` | The conversation and replies |
| `usePatchSupportTicketMutation` / `useCloseSupportTicketMutation` | Status changes |
| `usePlatformUsersQuery` | The user filter |
| `useGetMentorSettingsQuery` / `useEditMentorMutation` | Read and save `capability_human_support` |
| `useGetToolsQuery` | Whether the agent's model can use Human Support (`capability_field`, `is_allowed`): a "no" locks the switch while it is off |

All from `@iblai/iblai-js/data-layer`, with the `SupportTicket`,
`SupportTicketMessage` and `SupportTicketStatus` types. REST twin:
[`/iblai-api-agent-support`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-api-agent-support/SKILL.md).

## Step 5: Verify

Run `/iblai-vibe-ops-test` before telling the user the work is ready, and give
every page, control and state this skill added its own checkpoint — one Playwright
test titled `<ID> · …` in `e2e/journeys/`, listed in `e2e/coverage.json`;
`pnpm e2e:coverage` must pass:

1. `pnpm build` -- must pass with zero errors.
2. `pnpm test` -- vitest must pass.
3. Touch test `/agents/<uuid>/support` in the browser as `/iblai-vibe-ops-test` says, signed
   in as an org admin: the Human Support switch and the filters render; turn the switch
   on ("Human Support turned on"), ask the agent in chat to hand you to a human -- a
   ticket appears in the inbox; read the console; check the phone width.

## Important Notes

- **The switch is a flag**: it saves `capability_human_support` (plus
  `can_use_tools: true` when turning it on for an agent that could not use
  tools yet) and never rewrites the agent's tool list (the SDK's Tools tab is
  gone). Turning it off hides the hand-over from users; existing tickets stay
  workable.
- **Locked or refused**: the switch is locked, with the reason under the info
  box, when the person may not change the flag, or, while it is off and the
  agent could not use tools yet, may not change `can_use_tools` ("You don't
  have permission to change this."). While it is off it is also locked when
  the agent's model can't use it ("Doesn't work with the model this agent
  uses. Pick another model in the LLM tab to turn it on."); a switch already
  on can still be turned off. A refused save turns it back and shows one
  sentence there, never a toast.
- **Admins see everything**: org admins see every ticket for the agent;
  other users see only their own.
- **Shared provider**: mount `AgentSettingsProvider` once at the layout
  level (`/iblai-vibe-agent` §1); do not wrap each tab.
- **Peer deps**: `sonner` and `@iblai/iblai-web-mentor`
  (`pnpm add sonner @iblai/iblai-web-mentor`).
- **Brand guidelines**: [BRAND.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/BRAND.md)
