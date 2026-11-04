# Getting started — the whole journey

> Mirrored from [`iblai/vibe`](https://github.com/iblai/vibe) · [`docs/getting-started.md`](https://github.com/iblai/vibe/blob/main/docs/getting-started.md). This page is generated — edit it in the repository, not here.

The README gives you the three steps. This page is what actually happens in
each, which entry point is yours, and what to do when you are not starting from
scratch.

## Which entry point is yours

| You are | Architecture | Entry point |
|---|---|---|
| Starting a **new app for one organization** — yours, or a customer you deploy to (most apps) | **single-org**: the org key is pinned, one server-to-server token, members sign in with SSO | `/iblai-vibe-start` → `/iblai-vibe-ops-init` copies **vibe-starter** |
| Building **one deployment for many organizations**, users moving between them (the os.ibl.ai model) | **multi-org**: org in the URL, a switcher, re-auth per org | `/iblai-vibe-start` → `/iblai-vibe-ops-init`, then `/iblai-vibe-auth` → "Going multi-org" |
| Adding ibl.ai to an **existing Next.js app** | either of the above | `/iblai-vibe-start` → `/iblai-vibe-auth`, then the feature skills |
| Adding ibl.ai to an **existing app that is not Next.js** | either | `/iblai-vibe-start` → `/iblai-api-login`; the `iblai-api-*` skills work anywhere, the components need Next.js |
| Writing a **script, CI job, or backend** — nobody signs in | **headless**: a Platform API Token per org | `/iblai-vibe-start` → `/iblai-api-login`, then the `iblai-api-*` skill for the family |

How sign-in and organizations really work: [auth-model.md](/developer/vibe/auth-model).
The data every app is built on: [domain-model.md](/developer/vibe/domain-model).

## The journey

0. **Get an organization** — sign up at [ibl.ai/join](https://ibl.ai/join); it creates your account *and your organization*. (Everyone is also in the shared `main` org — that one is not yours.) → [Platform lifecycle](/developer/vibe/platform-lifecycle)
1. **Get two values** — your **org key** (on [login.iblai.app/me](https://login.iblai.app/me)) and a **Platform API Token** (os.ibl.ai → Admin → Integrations → APIs → Add API). → [same doc](/developer/vibe/platform-lifecycle#4-value-two-a-platform-api-token)
2. **Scaffold and run** — install the skills, run `/iblai-vibe-ops-init`, `pnpm dev`, sign in, pick or create your agent on `/setup`.
3. **Add the core** — profile, admin area, custom user and org data, memory, analytics, notifications: one skill each, [below](#the-core-app).
4. **Make it yours** — your pages with shadcn/ui; anything without a component through the REST API ([`/iblai-vibe-api`](/developer/skills/iblai-vibe-api)).
5. **Charge (optional)** — [`/iblai-vibe-pricing`](/developer/skills/iblai-vibe-pricing): how you are billed, and the three ways to bill your users.
6. **Ship** — [`/iblai-vibe-ops-deploy`](/developer/skills/iblai-vibe-ops-deploy) for a URL; [`/iblai-vibe-ops-build`](/developer/skills/iblai-vibe-ops-build) and [`/iblai-vibe-ops-release`](/developer/skills/iblai-vibe-ops-release) for desktop, mobile, and the stores.

Every step in detail, with what to click and what to check:
[platform-lifecycle.md](/developer/vibe/platform-lifecycle).

## Connecting your organization

`/iblai-vibe-connect` opens `login.iblai.app/connect`: you sign in (or create
an organization), pick the organization, click **Connect**, and the agent
receives the org key, your username, and a fresh Platform API Token on a
loopback callback — it writes `iblai.env`, `.env`, and `.env.local` and never
shows the token. No browser (SSH, a container)? The same page has a manual mode
you paste into the chat. Until the hosted page ships, the skills fall back to
asking for the two values and telling you exactly where they are:
[connect-flow.md](/developer/vibe/connect-flow).

## The core app

What `/iblai-vibe-ops-init` gives you, and the skill that owns each piece.
★ marks the six core data families almost every app reads or writes ([which scope stores what](/developer/skills#core)) — each links
its headless twin, the `iblai-api-*` skill that documents the same data over REST.

| You get | Skill | Reads / writes on the platform |
|---|---|---|
| SSO sign-in, session, org resolution (a placeholder or `main` org shows an alert, not a login loop) | [`/iblai-vibe-auth`](/developer/skills/iblai-vibe-auth) | [login](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/start/iblai-api-login/SKILL.md) |
| **Home = chat** with your agent — streaming, files, voice, canvas | [`/iblai-vibe-agent-chat`](/developer/skills/iblai-vibe-agent-chat) (+ [sidebar](/developer/skills/iblai-vibe-agent-chat-sidebar), [projects](/developer/skills/iblai-vibe-project)) | [agent-session](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/agents/iblai-api-agent-session/SKILL.md) |
| Agents browser (favorites, featured, custom, all) | [`/iblai-vibe-agent-search`](/developer/skills/iblai-vibe-agent-search) | [search](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/agents/iblai-api-search/SKILL.md) |
| ★ Create an agent from the app; set its identity, visibility, capabilities; every settings tab | [`/iblai-vibe-agent-create`](/developer/skills/iblai-vibe-agent-create), [`/iblai-vibe-agent-setting`](/developer/skills/iblai-vibe-agent-setting), [`/iblai-vibe-agent`](/developer/skills/iblai-vibe-agent) | ★ [agent-setting](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/agents/iblai-api-agent-setting/SKILL.md) |
| ★ The user's profile — name, bio, image, education, résumé | [`/iblai-vibe-profile`](/developer/skills/iblai-vibe-profile) | ★ [profile](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/users/iblai-api-profile/SKILL.md) |
| ★ **Custom data per user** — preferences, flags, onboarding, app state (no DB, no localStorage) | [`/iblai-vibe-user-metadata`](/developer/skills/iblai-vibe-user-metadata) | ★ [profile-metadata](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/users/iblai-api-profile-metadata/SKILL.md) |
| ★ **Custom data per organization** — settings, name, logos, support email | [`/iblai-vibe-org-metadata`](/developer/skills/iblai-vibe-org-metadata), [`/iblai-vibe-account`](/developer/skills/iblai-vibe-account) | ★ [org](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/organizations/iblai-api-org/SKILL.md) |
| **Users vs admins** — User/Admin view switch, admin area, invites, roles and policies | [`/iblai-vibe-admin`](/developer/skills/iblai-vibe-admin), [`/iblai-vibe-invite`](/developer/skills/iblai-vibe-invite), [`/iblai-vibe-rbac`](/developer/skills/iblai-vibe-rbac) | [management](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/users/iblai-api-management/SKILL.md), [rbac](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/users/iblai-api-rbac/SKILL.md), [invite](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/users/iblai-api-invite/SKILL.md) |
| ★ **Memory** — what agents remember about people (user-global, per agent, agent knowledge) | [`/iblai-vibe-memory-guide`](/developer/skills/iblai-vibe-memory-guide), [`/iblai-vibe-memory`](/developer/skills/iblai-vibe-memory), [`/iblai-vibe-agent-memory`](/developer/skills/iblai-vibe-agent-memory) | ★ [agent-memory](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/agents/iblai-api-agent-memory/SKILL.md) |
| ★ **Analytics** — usage, users, topics, transcripts, costs, audit, reports; org-wide or per agent | [`/iblai-vibe-analytics`](/developer/skills/iblai-vibe-analytics), [`/iblai-vibe-agent-audit`](/developer/skills/iblai-vibe-agent-audit), [`/iblai-vibe-history`](/developer/skills/iblai-vibe-history) | ★ [analytics](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/analytics/iblai-api-analytics/SKILL.md) |
| Notifications — bell, inbox, alerts, admin send | [`/iblai-vibe-notification`](/developer/skills/iblai-vibe-notification) | [notification](https://raw.githubusercontent.com/iblai/vibe/refs/heads/main/skills/users/iblai-api-notification/SKILL.md) |
| Navbar, onboarding flow, README | [`/iblai-vibe-navbar`](/developer/skills/iblai-vibe-navbar), [`/iblai-vibe-onboard`](/developer/skills/iblai-vibe-onboard), [`/iblai-vibe-readme`](/developer/skills/iblai-vibe-readme) | — |
| Anything with no component | [`/iblai-vibe-api`](/developer/skills/iblai-vibe-api) | any of the 51 `iblai-api-*` skills |
| Test before showing work (build, tests, a browser touch test); keep current | [`/iblai-vibe-ops-test`](/developer/skills/iblai-vibe-ops-test), [`/iblai-vibe-ops-upgrade`](/developer/skills/iblai-vibe-ops-upgrade) | — |

## How money works

- **You** run on prepaid, rechargeable **credits** with a hard ceiling (plans Free → Trial → Premium, opt-in auto-recharge, no per-seat fees). Add your own LLM keys for more models; put **spend caps** on the org, an agent, or one user on one agent.
- **Your users** can be charged three ways: pay-to-enter for the whole app on your own Stripe key with no commission ([`/iblai-vibe-monetization-app-paywall`](/developer/skills/iblai-vibe-monetization-app-paywall), the [vibe-agent](https://github.com/iblai/vibe-agent) model); per-item sales with subscriptions and revenue dashboards via Stripe Connect ([`/iblai-vibe-monetization`](/developer/skills/iblai-vibe-monetization) → [onboard](/developer/skills/iblai-vibe-monetization-onboard), [configure](/developer/skills/iblai-vibe-monetization-configure), [checkout](/developer/skills/iblai-vibe-monetization-checkout), [subscription](/developer/skills/iblai-vibe-monetization-subscription), [analytics](/developer/skills/iblai-vibe-monetization-analytics)); or reselling platform credits ([`/iblai-vibe-credit`](/developer/skills/iblai-vibe-credit), [`/iblai-vibe-billing`](/developer/skills/iblai-vibe-billing), [`/iblai-vibe-agent-billing`](/developer/skills/iblai-vibe-agent-billing)).
- One page decides: [`/iblai-vibe-pricing`](/developer/skills/iblai-vibe-pricing).

## Two families of skills, one install

Every skill carries a `kind` in its frontmatter (`ui`, `api`, `guide`, `ops`,
`security` — [docs/skill-kinds.md](/developer/vibe/skill-kinds)). The name prefix tells
you the family; the kind tells you whether there is a screen.

| | `iblai-vibe-*` — **build** (has a screen) | `iblai-api-*` — **operate** (no screen) |
|---|---|---|
| What it does | Mounts an SDK visual component in your Next.js app — a page, a tab, a dialog, a widget | Drives the platform headlessly: exact REST endpoints (method, URL, body, errors), `curl` first |
| Who it is for | Builders shipping an app | Anyone operating an organization — from a terminal, a script, CI, a server route, or an agent |
| Credentials | The signed-in user's session in the browser; `IBLAI_API_KEY` only in server routes | `IBLAI_API_KEY` (Platform API Token) as `Authorization: Api-Token`, from `.env` or `iblai.env` |
| Start | `/iblai-vibe` → `/iblai-vibe-ops-init` | `/iblai-api-login` |
| Also | `guide` (indexes and decision pages), `ops` (build, deploy, test, release), `security` | Tutorials in [`tutorials/`](https://github.com/iblai/vibe/tree/main/tutorials/); the hosted chat MCP server in [`mcp/`](https://github.com/iblai/vibe/tree/main/mcp/) |

Builders use both: the `ui` skill mounts the component, its **Platform data**
table names the `api` twin for anything the component doesn't cover, and
[`/iblai-vibe-api`](/developer/skills/iblai-vibe-api) is the server-route pattern
that joins them. The `iblai-api-*` family is the former
[`iblai/api`](https://github.com/iblai/vibe/tree/main/skills) repository, merged
here with its authoring contract intact — [docs/api-skills.md](/developer/vibe/api-skills).

### Headless API skills (`kind: api`)

Connect once with [`/iblai-api-login`](/developer/skills/iblai-api-login) (org key,
username, Platform API Token → `.env` + `iblai.env`), then any skill below is a
`/` command that calls `https://api.iblai.app` with `Authorization: Api-Token`.

| Area | Skills |
|---|---|
| **Setup** | [`/iblai-api-login`](/developer/skills/iblai-api-login) (start with [`/iblai-vibe-start`](/developer/skills/iblai-vibe-start) if you have not) |
| **One agent** (★ = the families most apps read or write) | ★ [`agent-setting`](/developer/skills/iblai-api-agent-setting), ★ [`agent-memory`](/developer/skills/iblai-api-agent-memory), [`agent-create`](/developer/skills/iblai-api-agent-create), [`agent-prompt`](/developer/skills/iblai-api-agent-prompt), [`agent-llm`](/developer/skills/iblai-api-agent-llm), [`agent-dataset`](/developer/skills/iblai-api-agent-dataset), [`agent-tool`](/developer/skills/iblai-api-agent-tool), [`agent-access`](/developer/skills/iblai-api-agent-access), [`agent-embed`](/developer/skills/iblai-api-agent-embed), [`agent-mcp`](/developer/skills/iblai-api-agent-mcp), [`agent-safety`](/developer/skills/iblai-api-agent-safety), [`agent-privacy`](/developer/skills/iblai-api-agent-privacy), [`agent-disclaimer`](/developer/skills/iblai-api-agent-disclaimer), [`agent-history`](/developer/skills/iblai-api-agent-history), [`agent-audit`](/developer/skills/iblai-api-agent-audit), [`agent-eval`](/developer/skills/iblai-api-agent-eval), [`agent-skill`](/developer/skills/iblai-api-agent-skill), [`agent-sandbox`](/developer/skills/iblai-api-agent-sandbox), [`agent-support`](/developer/skills/iblai-api-agent-support) |
| **Talk to an agent** | [`agent-session`](/developer/skills/iblai-api-agent-session) (REST/SSE/WebSocket), [`agent-chat`](/developer/skills/iblai-api-agent-chat) (hosted MCP server), [`inference`](/developer/skills/iblai-api-inference) (OpenAI-compatible `/v1`) |
| **Organization** | ★ [`org`](/developer/skills/iblai-api-org), [`management`](/developer/skills/iblai-api-management), [`rbac`](/developer/skills/iblai-api-rbac), [`invite`](/developer/skills/iblai-api-invite), [`scim`](/developer/skills/iblai-api-scim), [`token`](/developer/skills/iblai-api-token), [`integration`](/developer/skills/iblai-api-integration), [`notification`](/developer/skills/iblai-api-notification), [`feature`](/developer/skills/iblai-api-feature), [`billing`](/developer/skills/iblai-api-billing), [`spend-caps`](/developer/skills/iblai-api-spend-caps), [`crm`](/developer/skills/iblai-api-crm), [`external-service-proxy`](/developer/skills/iblai-api-external-service-proxy) |
| **The signed-in user** | ★ [`profile`](/developer/skills/iblai-api-profile), ★ [`profile-metadata`](/developer/skills/iblai-api-profile-metadata) |
| **Content, discovery, analytics** | ★ [`analytics`](/developer/skills/iblai-api-analytics), [`search`](/developer/skills/iblai-api-search), [`catalog`](/developer/skills/iblai-api-catalog), [`catalog-media`](/developer/skills/iblai-api-catalog-media), [`catalog-invitation`](/developer/skills/iblai-api-catalog-invitation), [`milestone`](/developer/skills/iblai-api-milestone), [`credential`](/developer/skills/iblai-api-credential), [`apply`](/developer/skills/iblai-api-apply) |
| **Other LMSs** | [`canvas-course-builder`](/developer/skills/iblai-api-canvas-course-builder) (Canvas REST API) |
| **Guides (no endpoints)** | [`ecosystem`](/developer/skills/iblai-api-ecosystem), [`infrastructure`](/developer/skills/iblai-api-infrastructure) (self-hosting) |

Every name above is `/iblai-api-<name>`.

The full, tiered catalogue with one-line descriptions is in `CLAUDE.md`.
Skills are in `skills/`; `npx skills add` installs them into your project's
`.claude/skills/`. Read them, extend them, or write your own — the front door
for agents is [`/iblai-vibe`](/developer/skills/iblai-vibe).

## The ibl.ai backend

`https://api.iblai.app` powers every vibe app; you build, host, and maintain
no backend services. It provides SSO (OAuth/OIDC/SAML) with multi-org
isolation and RBAC; agents with streaming, tools, RAG, memory, voice, and any
LLM; analytics; notifications; per-user and per-org metadata; credits,
spend caps, and Stripe billing. In the browser the SDK sends the signed-in
user's session token; your server holds one org-scoped Platform API Token in
the gitignored `iblai.env` / `.env.local`. How that fits together:
[docs/security-model.md](/developer/vibe/security-model).

### Two MCP servers

**`@iblai/mcp`** — SDK documentation for your coding assistant (components,
hooks, RTK Query endpoints, provider setup). `.mcp.json` at the project root
(vibe-starter ships it):

```json
{
  "mcpServers": {
    "iblai": { "command": "pnpm", "args": ["dlx", "@iblai/mcp"] }
  }
}
```

(`npx -y @iblai/mcp` in npm projects.) It gives your assistant
`get_component_info`, `get_hook_info`, `get_api_query_info`,
`get_provider_setup`, `create_page_template`, `get_playwright_helper_info`.

**`iblai-agent-chat`** — a *hosted* MCP server for the one runtime capability
that is not a REST call: holding a live conversation with a deployed agent
(streamed responses, tool use, RAG). No install; wire it with
[`/iblai-api-agent-chat`](/developer/skills/iblai-api-agent-chat) or by hand from
[`mcp/iblai-agent-chat/README.md`](https://github.com/iblai/vibe/blob/main/mcp/iblai-agent-chat/README.md):

```bash
claude mcp add iblai-agent-chat --transport http https://asgi.data.iblai.app/mcp/agent-chat/ \
  --header "Authorization: Api-Token YOUR_API_TOKEN" --header "X-Mentor-Unique-Id: YOUR_AGENT_UUID"
```
