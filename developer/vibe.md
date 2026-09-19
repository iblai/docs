# iblai/vibe

> Mirrored from [`iblai/vibe`](https://github.com/iblai/vibe) · [`README.md`](https://github.com/iblai/vibe/blob/main/README.md). This page is generated — edit it in the repository, not here.

Ship AI-powered apps fast — and run the platform behind them. Backend included.

[Release](https://github.com/iblai/vibe/releases/latest)
[Skills](https://github.com/iblai/vibe/blob/main/docs/skill-kinds.md)
[Next.js](https://nextjs.org)
[TypeScript](https://www.typescriptlang.org)
[Tailwind CSS](https://tailwindcss.com)
[Claude Code](https://claude.ai/code)
[OpenAI Codex](https://openai.com/codex)
[Cursor](https://cursor.com)
[Desktop & Mobile](https://github.com/iblai/vibe/blob/main/skills/ship/iblai-vibe-ops-build/SKILL.md)
[License: MIT](#license)

> **Note:** This toolkit runs against the hosted `iblai.app` environment. If you'd like a license to the full platform codebase to run locally or self-host, reach out to our team at [ibl.ai/contact](https://ibl.ai/contact).

---

Build the app you actually need on [ibl.ai](https://ibl.ai) — sign-in, an AI
agent to talk to, users and admins, custom data, memory, analytics, billing —
as a website and as macOS, Windows, iOS, and Android apps. Or run the platform
with no UI at all. One install; your coding agent does the rest.

## Three steps

**1. Install the skills** in the agent you already use.

```bash
npx skills add iblai/vibe --all        # Claude Code, OpenAI Codex, Cursor, OpenCode, Copilot …
```

Claude Code users can install the plugin instead (same skills, plus the SDK-docs
MCP server, updated with `/plugin update`): `/plugin marketplace add iblai/vibe`
then `/plugin install iblai-vibe@iblai`.

**2. Say hello.**

```text
/iblai-vibe-start
```

Four questions — new project or existing code · one organization, many, or none ·
who signs in · what it's about (users · memories · agents · organizations). Then
it connects your ibl.ai organization (one click in the browser, or two pasted
values — the token never enters the chat) and builds. Thirty seconds after you
sign in you are chatting with your own agent, with users, an admin area, custom
user and organization settings, memory, analytics, and billing already wired.

**3. Ship.** `/iblai-vibe-ops-deploy` gives you a URL. `/iblai-vibe-ops-build`
gives you macOS, Windows, iOS, and Android.

New to ibl.ai? [ibl.ai/join](https://ibl.ai/join) creates your account and your
organization — free to start. Want the details of every step?
[Getting started →](https://github.com/iblai/vibe/blob/main/docs/getting-started.md)

## What you get

vibe-starter, the app `/iblai-vibe-start` sets up for the common case:

- **Sign-in** with ibl.ai SSO, pinned to your organization
- **Home is a chat** with your agent; `/agents` to browse; `/setup` to pick or create one
- **Users and admins** — profile, a User/Admin switch, an admin area (people, invites, roles, analytics, billing, memory, organization)
- **Custom data** per user and per organization, stored on the platform — no database of your own
- **Memory, analytics, notifications, billing** — the platform's own, already on the page
- **Tests** (Vitest + Playwright) and a server helper for anything the components don't cover

Every piece is a skill you can also add to an existing app.

## Six things almost every app touches

Whatever you build, you will read or write these. Each has a component skill
and a headless twin that does the same thing over REST:

| | With a screen | Headless |
|---|---|---|
| **Custom data per user** — preferences, flags, onboarding, app state | `/iblai-vibe-user-metadata` | `/iblai-api-profile-metadata` |
| **The user's profile** — name, bio, image, résumé | `/iblai-vibe-profile` | `/iblai-api-profile` |
| **Memory** — user-global, per user × agent, agent knowledge | `/iblai-vibe-memory-guide` → `/iblai-vibe-memory`, `/iblai-vibe-agent-memory` | `/iblai-api-agent-memory` |
| **Custom data per organization** — branding, toggles, defaults | `/iblai-vibe-org-metadata` | `/iblai-api-org` |
| **Agent configuration** — create; identity, visibility, capabilities | `/iblai-vibe-agent-create` → `/iblai-vibe-agent-setting` | `/iblai-api-agent-create`, `/iblai-api-agent-setting` |
| **Analytics** — usage, transcripts, costs, audit, reports | `/iblai-vibe-analytics` | `/iblai-api-analytics` |

Not sure whether you want the per-user, per-agent, or per-organization one?
[Which scope stores what →](https://github.com/iblai/vibe/blob/main/docs/catalogue.md#core)

## The skills, by folder

Two families in one install: **🖥️ `iblai-vibe-*`** mounts a visual component in
your app; **🔌 `iblai-api-*`** does the same thing headlessly over REST — for a
script, a CI job, or your own backend. [How they differ →](https://github.com/iblai/vibe/blob/main/docs/skill-kinds.md)

| Folder | What it covers | Skills |
|---|---|---|
| [`start/`](https://github.com/iblai/vibe/tree/main/skills/start) | The first conversation, connecting your organization, sign-in, the starter | 11 |
| [`agents/`](https://github.com/iblai/vibe/tree/main/skills/agents) | Chat, browse, create, and configure agents — every settings tab | 56 |
| [`users/`](https://github.com/iblai/vibe/tree/main/skills/users) | Profiles, custom user data, memories, roles and admins, invitations | 18 |
| [`organizations/`](https://github.com/iblai/vibe/tree/main/skills/organizations) | Organization settings and metadata, branding, integrations | 9 |
| [`billing/`](https://github.com/iblai/vibe/tree/main/skills/billing) | How you are charged, spend caps, three ways to charge your users | 12 |
| [`analytics/`](https://github.com/iblai/vibe/tree/main/skills/analytics) | Usage, users, topics, transcripts, costs, reports | 2 |
| [`content/`](https://github.com/iblai/vibe/tree/main/skills/content) | Courses, catalog, credentials, admissions, Open edX Studio | 25 |
| [`ship/`](https://github.com/iblai/vibe/tree/main/skills/ship) | Test, deploy, native builds, app stores | 11 |
| [`security/`](https://github.com/iblai/vibe/tree/main/skills/security) | Authorized-use security work | 8 |

The full list with one line per skill: [docs/catalogue.md](https://github.com/iblai/vibe/blob/main/docs/catalogue.md).

## Keep it current

This repo changes often. Installed skills are a copy — refresh them with the
install command above, or `/iblai-vibe-ops-upgrade` in Claude Code (your agent
will suggest it when the copy is more than two weeks old).
[Details →](https://github.com/iblai/vibe/blob/main/docs/keep-current.md)

## Learn more

- [Getting started](https://github.com/iblai/vibe/blob/main/docs/getting-started.md) — the journey in detail, and what to do when you already have an app
- [How sign-in and organizations work](https://github.com/iblai/vibe/blob/main/docs/auth-model.md) — one organization, many, or none
- [Users, agents, organizations](https://github.com/iblai/vibe/blob/main/docs/domain-model.md) — the data every app is built on
- [How money works](https://github.com/iblai/vibe/blob/main/skills/billing/iblai-vibe-pricing/SKILL.md) · [Ship anywhere](https://github.com/iblai/vibe/blob/main/docs/ship.md) · [The two MCP servers](https://github.com/iblai/vibe/blob/main/docs/mcp-servers.md)
- [Contributing a skill](https://github.com/iblai/vibe/blob/main/CONTRIBUTING.md) — including where it goes and how to catalogue it

## Built with iblai/vibe

[Agentic OS](https://os.ibl.ai) ([iblai/os](https://github.com/iblai/os)) ·
[Agentic LMS](https://lms.ibl.ai) ([iblai/lms](https://github.com/iblai/lms)) ·
[vibe-agent](https://github.com/iblai/vibe-agent) (one creator, one agent, one paywall) ·
marketing skills in [iblai/vibe-marketing](https://github.com/iblai/vibe-marketing)

## License

MIT — [ibl.ai](https://ibl.ai)
