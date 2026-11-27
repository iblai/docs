# /iblai-vibe-agent-dataset

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-vibe-agent-dataset/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-vibe-agent-dataset/SKILL.md). This page is generated — edit it in the repository, not here.

Add the agent **Datasets tab** -- the agent's knowledge base. **Scope: per
agent.** Each resource is a training document attached to the agent, and the
agent answers from every trained document for every user who chats with it.
The tab is a searchable table (five rows per page) with an **Add Resource**
button that opens the SDK's own resource picker. This is one tab in the
agent-settings family indexed by `/iblai-vibe-agent`; it shares the
`AgentSettingsProvider` wrapper with every other tab.

**Datasets** -- name (links to the source), type, tokens, retrain interval,
visibility, and a training switch. **Add Resource** shows only when the user
holds the create grant on the agent's documents (Step 2).

![Datasets tab -- table](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-dataset/iblai-vibe-agent-dataset-1-datasets.png)

**Enhanced Document Retrieval** -- since `@iblai/iblai-js` 3.0.0 a box right
under the info box, above the search row
(`capability-card-advanced-document-search`), holds this switch, next to the
documents it searches. The sentence under its label says what it does: "Let
your agent search its documents more thoroughly. It runs several searches for
each question and can search again while it works on a reply." One click
saves `capability_advanced_document_search` and `enable_multi_query_rag` to
the same value in one request, on the click. The switch shows the flag, so an
agent that has only `enable_multi_query_rag` on shows off; either click brings
the two in step. It replaces the Enhanced Document Retrieval row that Settings
› Capabilities had.

When the agent's model cannot use the document search tool, the switch stays
usable for `enable_multi_query_rag` alone, which works with any model. It then
shows and saves only that field, and says "With this model, your agent can't
search again while it works on a reply." under its description
(`capability-limited-advanced-document-search`). Once the flag is on, the
switch works as usual, so it can be turned off. Locks, refused saves and the
`can_use_tools: true` it adds when needed work as for every capability switch
(`/iblai-vibe-agent-tool`). The box renders nothing when the user may not read
the flag or the platform is older than API version 4.467.0.

**Add Resources** -- one tile per source: PowerPoint, OneDrive, Google Drive,
Dropbox, YouTube, URL, PDF, DOCX, Excel, CSV, GitHub, TXT, Markdown, Audio,
Video, Image, Web Crawler, ZIP, Course. `disabledResourceTypes` greys tiles out
and makes them unclickable; it does not hide them.

**Google Drive** -- the tile opens Google's own file picker over Add
Resources. Clicking outside Google's box or pressing Escape closes only
Google's box. When the browser blocks Google's cookies (Firefox Total Cookie
Protection, Brave Shields), Google first shows **Allow cookies**: clicking it
brings up the browser's Allow / Block prompt. If the picker has not loaded
after 10 seconds, Add Resources shows one red line, "Your browser blocked
Google Drive. Try again and click Allow cookies in the Google window.", with
a **Try Again** button that reopens it.

![Datasets tab -- Add Resources](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-dataset/iblai-vibe-agent-dataset-2-add-resource.png)

**Source dialog** -- each tile opens its own dialog (here **URL**). **Submit**
queues the resource ("Document has been queued for training"); the row shows
"In progress" until training finishes.

![Datasets tab -- URL dialog](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-dataset/iblai-vibe-agent-dataset-3-url.png)

**Schedule Retraining** -- the clock in **Interval** opens a retrain schedule
(daily, weekly, monthly, or a custom number of days). It is enabled only for
trained documents that can be re-fetched (URLs, YouTube, GitHub, crawls);
uploaded files and cloud-drive documents cannot be retrained.

![Datasets tab -- Schedule Retraining](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-dataset/iblai-vibe-agent-dataset-4-retrain.png)

**Untrain, then delete** -- switching a trained row off untrains it and then
offers **Delete Dataset**. Switching an untrained row on asks whether to
**Train** it (web crawls take an optional User-Agent) or **Delete** it.

![Datasets tab -- Delete Dataset](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-dataset/iblai-vibe-agent-dataset-5-delete.png)

> **Common setup (brand, conventions, env files, verification):** see [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/docs/skill-setup.md).

## Prerequisites

- Auth set up (`/iblai-vibe-auth`, or vibe-starter).
- `AgentSettingsProvider` wraps the route (`/iblai-vibe-agent` §1).
- `@iblai/iblai-js@^3.1.0` (resolves `@iblai/web-containers` 2.1.0). Check with
  `pnpm why @iblai/web-containers`. The Google Drive behaviour below (Allow
  cookies, Try Again) needs `@iblai/iblai-js` 2.62.2 or later.
- A real agent UUID. Ask the user; never invent one.
- For the Google Drive tile: the organization's `drive` credential, added
  under organization settings → Secrets → **Services** (`client_id`,
  `developer_key`; REST in
  [`/iblai-api-integration`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/organizations/iblai-api-integration/SKILL.md)).
  The `client_id` must be a Google OAuth client id of the form
  `<project-number>-<x>.apps.googleusercontent.com` with the `drive.file`
  scope: the picker sends the leading number (the Cloud project number) as
  `appId`, which lets the app open the files a user picks.

## Step 1: Check Environment

Look for `iblai.env` in the project root with `PLATFORM`, `DOMAIN`, and
`TOKEN`. If it is missing, tell the user:
"You need an `iblai.env` with your platform configuration. Download the
template and fill in your values:
`curl -o iblai.env https://raw.githubusercontent.com/iblai/base/refs/heads/main/iblai.env`"

## Step 2: Mount `AgentDatasetsTab`

**Add Resource** is gated on the RBAC grant
`/mentors/<agent db id>/documents/#create`, read from the provider's
`rbacPermissions` -- even with `enableRBAC={false}`. Without the grant the
table renders but the button never shows. Load the grants for the agent and
pass them to a nested provider:

```tsx
// app/(app)/agents/[mentorId]/datasets/page.tsx
"use client";

import { useEffect, useState } from "react";
import {
  AgentDatasetsTab,
  AgentSettingsProvider,
  useAgentSettings,
} from "@iblai/iblai-js/web-containers/next";
import {
  useGetMentorSettingsQuery,
  useGetRbacPermissionsMutation,
} from "@iblai/iblai-js/data-layer";

export default function AgentDatasetsPage() {
  const settings = useAgentSettings();
  const { tenantKey, mentorId, username } = settings;
  const [grants, setGrants] = useState<object>({});
  const [getRbacPermissions] = useGetRbacPermissionsMutation();
  const { data } = useGetMentorSettingsQuery({
    mentor: mentorId,
    org: tenantKey,
    // @ts-expect-error userId is accepted at runtime
    userId: username,
  });
  const mentorDbId = (data as { mentor_id?: number } | undefined)?.mentor_id;

  useEffect(() => {
    if (!mentorDbId) return;
    getRbacPermissions({
      requestBody: {
        platform_key: tenantKey,
        resources: [`/mentors/${mentorDbId}/documents/`],
      },
    })
      .unwrap()
      .then((permissions) => setGrants({ ...permissions }))
      .catch(() => {});
  }, [mentorDbId, tenantKey, getRbacPermissions]);

  return (
    <div className="flex h-full flex-col bg-white">
      <AgentSettingsProvider {...settings} rbacPermissions={grants}>
        <AgentDatasetsTab maxUploadSizeMb={50} />
      </AgentSettingsProvider>
    </div>
  );
}
```

The grant path uses the agent's numeric `mentor_id` from its settings, not
the UUID. If your layout already loads grants into the provider, mount
`<AgentDatasetsTab />` directly.

### Keep page and search in the URL (optional)

By default the tab keeps page and search in local state. To survive reloads
and share links, control them:

```tsx
"use client";

import { useRouter, useSearchParams } from "next/navigation";
import { AgentDatasetsTab } from "@iblai/iblai-js/web-containers/next";

export function UrlSyncedDatasets() {
  const router = useRouter();
  const params = useSearchParams();
  const page = Number(params.get("page") ?? "1") || 1;
  const search = params.get("q") ?? "";

  const replace = (next: Record<string, string | null>) => {
    const sp = new URLSearchParams(params.toString());
    for (const [key, value] of Object.entries(next)) {
      if (value) sp.set(key, value);
      else sp.delete(key);
    }
    router.replace(`?${sp.toString()}`);
  };

  return (
    <AgentDatasetsTab
      page={page}
      search={search}
      onPageChange={(p) => replace({ page: String(p) })}
      onSearchChange={(q) => replace({ q: q || null, page: null })}
    />
  );
}
```

`onSearchChange` implies a reset to page 1: drop the page param in it.

## Step 3: Customize Labels (Optional)

The tab renders with the default agent-facing copy
(`AGENT_DATASETS_TAB_LABELS`, localized through the SDK's i18n). Pass a
partial `labels` object to change any string:

```tsx
import { AgentDatasetsTab } from "@iblai/iblai-js/web-containers/next";

<AgentDatasetsTab
  labels={{
    header: { title: "Knowledge base" },
    addResource: { button: "Add source" },
  }}
/>;
```

Label groups (`DatasetsTabLabels`): `header` (`title`, `description`),
`infoBox`, `search.placeholder`, `addResource.button`, `table` (the six
column headings and `emptyState`), `toasts` (`updateSuccess`, `updateError`,
`deleteSuccess`, `deleteError`, `trainQueued`, `retrainSuccess`,
`retrainError`), `deleteModal`, `trainOrDeleteModal` (including the crawler
`userAgent*` copy), and `retrainScheduleModal`. The Add Resources dialogs use
the SDK's own copy, and so does the Enhanced Document Retrieval box: its name,
sentence, model note, lock reasons, toasts and errors come from the SDK's own
catalogue, not from `labels`.

## Step 4: Use MCP Tools for Customization

```
get_component_info("AgentDatasetsTab")
get_component_info("AgentSettingsProvider")
```

## `<AgentDatasetsTab>` Props

Import from `@iblai/iblai-js/web-containers/next`.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `labels` | `DeepPartial<DatasetsTabLabels>` | No | Override user-visible strings |
| `disabledResourceTypes` | `string[]` | No | Tile ids to disable (greyed out, still shown) in Add Resources (`powerpoint`, `onedrive`, `google-drive`, `dropbox`, `youtube`, `url`, `pdf`, `docx`, `excel`, `csv`, `github`, `text`, `markdown`, `audio`, ...) |
| `maxUploadSizeMb` | `number` | No | Upload size limit for local files |
| `dropboxExtensions` | `string[]` | No | File extensions the Dropbox picker offers |
| `onSelect` | `(dataset: Dataset) => void` | No | Picker mode: called when a row is clicked |
| `showCapabilitySwitch` | `boolean` | No | Show the Enhanced Document Retrieval box. Defaults to `true`, and to `false` with `onSelect`; pass `false` where the agent is still being set up (the SDK's Create Agent wizard does) |
| `selectedDatasetId` | `string` | No | Picker mode: highlight this row |
| `page` | `number` | No | Controlled page (1-based). Omit for internal paging |
| `onPageChange` | `(page: number) => void` | No | Fired with the requested page when `page` is controlled |
| `search` | `string` | No | Controlled search text. Omit for internal search |
| `onSearchChange` | `(search: string) => void` | No | Fired with the debounced search; reset the page to 1 here |
| `AddResourceModal` | `ComponentType<{ isOpen, onClose, keepParentOpen? }>` | No | **Deprecated.** Replaces the built-in Add Resources dialog |
| `PaginationComponent` | `ComponentType<{ currentPage, totalPages, onPageChange, disabled }>` | No | **Deprecated.** Replaces the built-in pagination |

## Related Exports

From `@iblai/iblai-js/web-containers/next`:

- `AGENT_DATASETS_TAB_LABELS` -- the default label bundle.
- `AgentDatasetsTabProps`, `DatasetsTabLabels`, `Dataset` -- types.

## How it saves

| Action | Request |
|---|---|
| Load | `GET documents/pathways/{uuid}/?limit=5&offset=…&search=…` |
| Add a resource | `POST documents/train/` (`multipart/form-data`, `pathway` = agent UUID, `type` per source) |
| Training switch off / on | `PUT documents/{id}/` with `train: false` / `train: true` (plus `crawler_extra_headers` when a User-Agent is set) |
| Visibility (eye icon) | `PUT documents/{id}/` with `access: "public"` or `"private"` |
| Schedule Retraining | `POST documents/{id}/settings/` with `retrain_interval_days` |
| Delete | `DELETE documents/{id}/` |
| Enhanced Document Retrieval | `PUT /api/ai-mentor/orgs/{org}/users/{username}/mentors/{uuid}/settings/` with `capability_advanced_document_search` and `enable_multi_query_rag` (plus `can_use_tools: true` when turning it on for an agent that could not use tools yet; only `enable_multi_query_rag` while the model limits it) |

The document paths sit under `/api/ai-index/orgs/{org}/users/{username}/`.
Test ids for the switch: `capability-card-advanced-document-search`,
`capability-toggle-advanced-document-search` (a `role="switch"` named
"Enhanced Document Retrieval"), `capability-error-advanced-document-search`,
`capability-reason-advanced-document-search` and
`capability-limited-advanced-document-search`.

## Platform data

| Hook | Purpose |
|---|---|
| `useGetTrainingDocumentsQuery` | The paged document list |
| `useEditTrainingDocumentMutation` | Train, untrain, visibility |
| `useGetMentorSettingsQuery` | The agent's `mentor_id` for the RBAC grant; the switch's flag and `permissions` |
| `useGetToolsQuery` | Whether the agent's model can use the document search tool (`capability_field`, `is_allowed`) |
| `useEditMentorMutation` | Save the Enhanced Document Retrieval switch |

REST twin: [`/iblai-api-agent-dataset`](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-api-agent-dataset/SKILL.md).

## Step 5: Verify

Run `/iblai-vibe-ops-test` before telling the user the work is ready, and give
every page, control and state this skill added its own checkpoint — one Playwright
test titled `<ID> · …` in `e2e/journeys/`, listed in `e2e/coverage.json`;
`pnpm e2e:coverage` must pass:

1. `pnpm build` -- must pass with zero errors.
2. `pnpm test` -- vitest must pass.
3. Touch test `/agents/<uuid>/datasets` in the browser as `/iblai-vibe-ops-test` says: the
   table renders and **Add Resource** shows; add a URL, wait for its switch to turn on,
   then switch it off and **Delete** it; flip **Enhanced Document Retrieval**, reload --
   it keeps its state -- and flip it back; read the console; check the phone width.

## Important Notes

- **Per agent, not per user**: every trained document feeds every chat with
  the agent.
- **No Add Resource button?** The grant is missing: load
  `/mentors/<mentor_id>/documents/` permissions (Step 2).
- **The switch saves on the click**, not with the table: it changes the
  agent's settings, so it applies to every chat with the agent.
- **Shared provider**: mount `AgentSettingsProvider` once at the layout
  level (`/iblai-vibe-agent` §1); the nested provider in Step 2 only adds
  grants.
- **Peer deps**: `sonner` and `@iblai/iblai-web-mentor`
  (`pnpm add sonner @iblai/iblai-web-mentor`).
- **Brand guidelines**: [BRAND.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/BRAND.md)
