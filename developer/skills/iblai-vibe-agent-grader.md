# /iblai-vibe-agent-grader

> Mirrored from [`iblai/base`](https://github.com/iblai/base) · [`skills/agents/iblai-vibe-agent-grader/SKILL.md`](https://github.com/iblai/base/blob/main/skills/agents/iblai-vibe-agent-grader/SKILL.md). This page is generated — edit it in the repository, not here.

Add the agent **Grader tab** -- let the agent score work against a rubric you
define. **Scope: per agent.** The **Grading** switch at the top saves the
agent's `capability_grading` flag. Under it, the **Lesson Completion** row
saves `capability_lesson_completion`: the agent marks a lesson complete once
the user finishes it, in new chats with signed-in users, where the agent is
added to a course lesson. That row works with Grading on or off.
Three sub-tabs follow, all usable while Grading is off: **Grading Setup**
(what gets graded, how much feedback is shared, the grading instructions),
**Rubric** (criteria and points), and **Results** (every grade the agent
issued, with overrides pushed back to the LMS). This is one tab in the
agent-settings family indexed by `/iblai-vibe-agent`; it shares the
`AgentSettingsProvider` wrapper with every other tab.

**Set up first, then switch on.** While Grading is on with no setup or an
empty rubric, the platform refuses every chat turn with the agent. So the
Grading switch stays off and locked until the rubric has a criterion, with
"Add at least one rubric criterion first, then turn Grading on." under it.
The order is: save the setup, add a criterion, turn Grading on. Grades always
show in Results; they reach a course only where the agent is added to a
course lesson.

**Grading Setup** -- **What Gets Graded** (A Submission / The Conversation),
**Feedback Shared with the Person** (Overall Feedback Only / Feedback per
Criterion / Overall + per Criterion), and the required **Grading
Instructions**. The first **Save** creates the setup; later saves update it.
An amber warning shows while Grading is on but the setup or the rubric is
missing (Grading turned on through the API, say): the agent can't answer
anyone until both are in place or Grading is off.

![Grader tab -- Grading Setup](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-grader/iblai-vibe-agent-grader-1-setup.png)

**Rubric** -- one row per criterion (Name, Criteria, Points) with an
**Edit** / **Delete** menu. The footer shows the total; the overall score is
points earned out of that total. **Add Criterion** is enabled once the setup
is saved; until then the rubric says "Save your grading setup first, then
add rubric criteria here."

![Grader tab -- Rubric](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-grader/iblai-vibe-agent-grader-2-rubric.png)

**Add Criterion** -- Name, Criteria (what earns the points), and Points (a
positive number). **Edit** opens the same dialog filled in.

![Grader tab -- Add Criterion](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-grader/iblai-vibe-agent-grader-3-criterion-add.png)

**Delete Criterion** -- asks for confirmation. The last criterion cannot be
deleted (its **Delete** is disabled); add another first.

![Grader tab -- Delete Criterion](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-grader/iblai-vibe-agent-grader-4-criterion-delete.png)

**Results** -- filter by user (the History tab's user list), status
(Pending / Published / Failed) and date range; 10 rows per page with
columns **Learner**, **Score** (percent), **Status**, **Override** and **Graded**. Each row has an
**Override** button that opens **Override Grade**: the AI score, **Override
Points** (0 up to the rubric total, capped at 100) and optional **Override
Feedback**; **Clear** restores the AI score. (The shot shows the empty
state: this agent had not graded anything yet.)

![Grader tab -- Results](https://raw.githubusercontent.com/iblai/base/refs/heads/main/skills/agents/iblai-vibe-agent-grader/iblai-vibe-agent-grader-5-results.png)

> **Common setup (brand, conventions, env files, verification):** see [docs/skill-setup.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/docs/skill-setup.md).

## Prerequisites

- Auth set up (`/iblai-vibe-auth`, or vibe-starter).
- `AgentSettingsProvider` wraps the route (`/iblai-vibe-agent` §1).
- `@iblai/iblai-js@^3.1.0` (resolves `@iblai/web-containers` 2.1.0). Check with
  `pnpm why @iblai/web-containers`.
- A real agent UUID. Ask the user; never invent one.
- A platform that returns the `capability_*` flags on the agent's settings
  (API version 4.467.0 or later). On an older one the Grading switch stays
  off and locked, and the Lesson Completion row does not render.

## Step 1: Check Environment

Look for `iblai.env` in the project root with `PLATFORM`, `DOMAIN`, and
`TOKEN`. If it is missing, tell the user:
"You need an `iblai.env` with your platform configuration. Download the
template and fill in your values:
`curl -o iblai.env https://raw.githubusercontent.com/iblai/base/refs/heads/main/iblai.env`"

## Step 2: Mount `AgentGraderTab`

```tsx
// app/(app)/agents/[mentorId]/grader/page.tsx
"use client";

import { AgentGraderTab } from "@iblai/iblai-js/web-containers/next";

export default function AgentGraderPage() {
  return (
    <div className="flex h-full flex-col bg-white">
      <AgentGraderTab />
    </div>
  );
}
```

The tab reads `tenantKey`, `mentorId`, `username`, `enableRBAC` and
`rbacPermissions` from `AgentSettingsProvider`; each can be passed as a prop
instead. With `enableRBAC={false}` (the `/iblai-vibe-agent` layout) every
grader action (setup, rubric, results) shows and the server's 403s are the
only gate -- no grants need loading. The two switches ignore `enableRBAC`:
they follow `permissions.field` on the agent's settings. A switch whose flag
the person may not change is locked; if they may not read `capability_grading`
the Grading switch is locked too, and if they may not read
`capability_lesson_completion` its row does not render. A 403 on the setup or
rubric replaces the tab body (the header stays) with "No Access to Grading
Settings".

With `enableRBAC` on, each action is checked as `/mentors/<agent db id>/#<action>`:
`read_grader_config`, `create_grader_config`, `write_grader_config`,
`view_grader_criteria`, `create_grader_criteria`, `write_grader_criteria`,
`delete_grader_criteria`, `view_grade_results`, `override_grade_results`.
Load them with `useGetRbacPermissionsMutation` (`requestBody: { platform_key,
resources: ["/mentors/<agent db id>/"] }`; the db id is `mentor_id` on the
agent's settings) and pass the result as `rbacPermissions`, on the provider or
as the tab's prop. The tab reads only that, never the store's `state.rbac`.
If the grants hold no entry for the agent, every action stays visible.

**How the two switches behave.** Each one saves on the click and toasts
"Grading turned on" / "Grading turned off — your rubric is kept for next
time" (Lesson Completion: "Lesson Completion turned on" / "Lesson Completion
turned off").
A switch is locked, with the reason under it, when the person may not change
its flag ("You don't have permission to change this.") or the agent's model
can't use it ("Doesn't work with the model this agent uses. Pick another
model in the LLM tab to turn it on."). A refused save turns the switch back
and shows one sentence under it, never a toast.

## Step 3: Customize Labels (Optional)

The tab renders with the default agent-facing copy
(`AGENT_GRADER_TAB_LABELS`, localized through the SDK's i18n). Pass a partial
`labels` object to change any string:

```tsx
import { AgentGraderTab } from "@iblai/iblai-js/web-containers/next";

<AgentGraderTab
  labels={{
    header: { title: "Auto-grading" },
    subTabs: { rubric: "Scoring rubric" },
  }}
/>;
```

Label groups (`GraderTabLabels`): `header` (`title`, `description`),
`capability` (the Grading switch: `title`, `description`, `offHint`,
`needsCriterion`), `subTabs`
(`setup`, `rubric`, `results`), `denied` (the no-access notice), `warnings`
(`noConfig`, `noCriteria`), `config` (the setup form: `gradingMode`,
`feedbackMode`, `instructions`, save buttons), `criteria` (the rubric table,
`totalPoints`, `scoreHint`, `lastCriterionHint`, `modal`, `deleteModal`),
`results` (`filters`, `statusValues`, `columns`, empty states,
`overrideModal`, `toasts`), and `toasts` (switch, setup and criterion
toasts; `toggleError` is no longer shown). The Lesson Completion row's name
and sentence come from the SDK's own catalogue, not from `labels`.

## Step 4: Use MCP Tools for Customization

```
get_component_info("AgentGraderTab")
get_component_info("AgentSettingsProvider")
```

## `<AgentGraderTab>` Props

Import from `@iblai/iblai-js/web-containers/next`.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `labels` | `DeepPartial<GraderTabLabels>` | No | Override user-visible strings |
| `tenantKey` | `string` | No | Organization key; defaults to `AgentSettingsProvider` |
| `mentorId` | `string` | No | Agent UUID; defaults to `AgentSettingsProvider` |
| `username` | `string` | No | Signed-in username; defaults to `AgentSettingsProvider` |
| `enableRBAC` | `boolean` | No | Check each grader action against `rbacPermissions`; defaults to `AgentSettingsProvider` |
| `rbacPermissions` | `object` | No | Permission tree; defaults to `AgentSettingsProvider` |

## Related Exports

From `@iblai/iblai-js/web-containers/next`:

- `AGENT_GRADER_TAB_LABELS` -- the default label bundle.
- `AgentGraderTabProps`, `GraderTabLabels` -- types.

`GRADING_TOOL_NAME` was removed in `@iblai/web-containers` 2.0.0: the tab
reads `capability_grading` now. Delete any import of it.

From `@iblai/iblai-js/data-layer`, for a custom UI on the same endpoints:
the hooks under Platform data, plus the types `MentorGraderConfiguration`,
`GradingMode` (`submission` | `conversation`), `FeedbackMode` (`overall` |
`per_criteria` | `both`), `GradeResult`, `GradeResultPublishStatus`
(`pending` | `published` | `failed`), `CreateMentorGraderConfigurationRequest`,
`CreateGraderCriterionRequest`, `UpdateGraderCriterionRequest`.

## How it saves

Paths are under `https://api.iblai.app/dm/api/ai-mentor/orgs/{org}/`. The
grader paths use the canonical `agents/` prefix; the tab calls the
`mentors/` alias for the setup and rubric, and both are served.

| Action | Request |
|---|---|
| **Grading** switch | `PUT users/{username}/mentors/{uuid}/settings/` with `capability_grading` only (plus `can_use_tools: true` when turning it on for an agent that could not use tools yet; `false` is never sent) |
| **Lesson Completion** switch | The same `PUT` with `capability_lesson_completion` only (same `can_use_tools` rule) |
| **Save** the setup | `POST agents/{uuid}/grader-config/` when the setup's `GET` answers 404, `PATCH` otherwise, with `grading_mode`, `feedback_mode`, `grader_instructions` (a `POST` when one exists returns 400 "already exists") |
| Add / edit / delete a criterion | `POST agents/{uuid}/grader-config/criteria/`; `PATCH` / `DELETE …/criteria/{id}/` (`name`, `criteria`, `points`) |
| Results | `GET agents/{uuid}/grader-results/?email=&status=&graded_at_after=&graded_at_before=&page=&page_size=10` |
| **Override** / **Clear** | `PATCH agents/{uuid}/grader-results/{id}/` with `override_points` (or `null` to clear) and `override_feedback` |

A custom UI saves the setup, adds at least one criterion, and only then sets
`capability_grading: true`. Turning grading off keeps the setup and rubric;
there is no way to delete the setup.

## Platform data

| Hook | Purpose |
|---|---|
| `useGetMentorSettingsQuery` / `useEditMentorMutation` | Read and save the two flags |
| `useGetToolsQuery` | Which capabilities the agent's model can use (`capability_field`, `is_allowed`): a "no" locks the switch |
| `useGetMentorGraderConfigurationQuery`, `useCreateMentorGraderConfigurationMutation`, `useUpdateMentorGraderConfigurationMutation` | The setup |
| `useListGraderCriteriaQuery`, `useCreateGraderCriterionMutation`, `useUpdateGraderCriterionMutation`, `useDeleteGraderCriterionMutation` | The rubric |
| `useListGradeResultsQuery`, `useGetGradeResultQuery`, `useOverrideGradeResultMutation` | Results and overrides |
| `useGetChatHistoryFilterQuery` | The Results user filter |

There is no `iblai-api-*` REST twin for the grader yet; the endpoints above
are in the live schema.

## Step 5: Verify

Run `/iblai-vibe-ops-test` before telling the user the work is ready, and give
every page, control and state this skill added its own checkpoint — one Playwright
test titled `<ID> · …` in `e2e/journeys/`, listed in `e2e/coverage.json`;
`pnpm e2e:coverage` must pass:

1. `pnpm build` -- must pass with zero errors.
2. `pnpm test` -- vitest must pass.
3. Touch test `/agents/<uuid>/grader` in the browser as `/iblai-vibe-ops-test` says, signed
   in as an org admin: the Grading switch, the Lesson Completion row and the three
   sub-tabs render; on a fresh agent the Grading switch is locked with the
   needs-criterion sentence; save the setup, add a criterion, turn Grading on -- "Grading
   turned on"; flip Lesson Completion; read the console; check the phone width.

## Important Notes

- **Activation is a flag**: the switch saves `capability_grading` (plus
  `can_use_tools: true` when turning it on for an agent that could not use
  tools yet); it never sends the agent's tool list (`tool_slugs`). The SDK's
  Tools tab is gone (`AgentToolsTab` was removed in `@iblai/web-containers`
  2.0.0).
- **Model changes**: while Grading is on, the platform refuses a model that
  can't use it, and the LLM tab's toast says "Grading is on, and the model
  you picked can't use it. Turn it off in the Grader tab first, or pick a
  different model."
- **Overrides reach the LMS**: an override is pushed back to the LMS the
  grade came from, as points ÷ rubric total.
- **Redux store**: the grading endpoints ship inside `mentorReducer` /
  `mentorMiddleware`, which vibe-starter's store already includes.
- **Setup is permanent**: it survives turning grading off and cannot be
  deleted. Try it on a test agent.
- **Shared provider**: mount `AgentSettingsProvider` once at the layout
  level (`/iblai-vibe-agent` §1); do not wrap each tab.
- **Peer deps**: `sonner` and `@iblai/iblai-web-mentor`
  (`pnpm add sonner @iblai/iblai-web-mentor`).
- **Brand guidelines**: [BRAND.md](https://raw.githubusercontent.com/iblai/base/refs/heads/main/BRAND.md)
