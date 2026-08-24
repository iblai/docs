# Vibe (`iblai/vibe`)

> **GitHub: [github.com/iblai/vibe](https://github.com/iblai/vibe)** — Ship AI-powered apps fast. Backend included. MIT license · Next.js · TypeScript · Tailwind CSS · desktop & mobile via Tauri v2.

## Overview

Vibe is a developer toolkit for **vibe coding new applications on top of the existing ibl.ai backend**. It gives you a production-ready scaffold powered by `iblai-app-cli`, the [@iblai/iblai-js](https://www.npmjs.com/package/@iblai/iblai-js) SDK, pre-built components, Claude Code skills, and a full backend at `iblai.app` — you go from zero to a deployed AI app in minutes, with authentication, AI chat, profiles, notifications, and analytics already wired up.

In the five-repo family, Vibe is the **build layer**: `iblai/api` operates the platform headlessly via its REST API, `iblai/os` and `iblai/lms` are complete sample applications built with this very toolkit, and `iblai/iblai-infra-cli` deploys the backend on your own infrastructure. Vibe is where you start when you want to create something new against that backend.

It is for developers (and their AI coding agents) who want a working app without building or hosting backend services. Auth is client-side via SSO — no API tokens to store, rotate, or leak. The toolkit runs against the hosted `iblai.app` environment; for a license to the full platform codebase to run locally or self-host, contact [ibl.ai/contact](https://ibl.ai/contact).

## What You Get

| Feature | Description |
|---------|-------------|
| **Authentication** | SSO login via iblai.app — no token management, session handling built in |
| **AI Chat** | Streaming chat with ibl.ai agents, markdown rendering, conversation history |
| **User Profile** | Editable profile page with avatar, bio, and preferences |
| **Account Settings** | Password changes, notification preferences, connected services |
| **Analytics Dashboard** | Usage metrics, conversation stats, and user activity |
| **Notifications** | Real-time notification system with read/unread state |
| **Desktop & Mobile** | Tauri v2 integration for macOS, Windows, Linux, iOS, and Android |
| **AI Development Skills** | Claude Code skills that walk you through adding and customizing every feature |

#### Built with iblai/vibe

Two production apps are built with this toolkit: **Agentic OS** ([os.ibl.ai](https://os.ibl.ai), repo [iblai/os](https://github.com/iblai/os)) and **Agentic LMS** ([lms.ibl.ai](https://lms.ibl.ai), repo [iblai/lms](https://github.com/iblai/lms)).

## How It Works

1. **Scaffold** — generate a full Next.js app with the CLI.
2. **Connect** — use Claude Code skills to add auth, AI chat, profiles, and more components that connect to `iblai.app` (or your own instance) for authentication, AI agents, and data.
3. **Customize** — use the skills to add features, swap components, and adjust business logic.
4. **Deploy** — push to Vercel or package with Tauri.

## Skills

After installing (`npx skills add iblai/vibe`), use the skills directly in your AI agent with `/` commands. Instead of reading docs, you tell your agent what you want and the skills provide the context.

#### App-building skills

- `/iblai-vibe-auth` — add SSO authentication and configure the app for ibl.ai login
- `/iblai-vibe-agent-chat` — add the full in-process agent chat surface
- `/iblai-vibe-agent-chat-sidebar` — wrap chat with the SDK sidebar (projects, pinned and recent messages)
- `/iblai-vibe-project` — add the in-process Projects surface (project landing page with chat input, files, instructions, assigned agents)
- `/iblai-vibe-profile` — add profile UI and profile settings flows
- `/iblai-vibe-history` — add the profile History surface (a user's own conversations, with filters and exports)
- `/iblai-vibe-account` — add account and organization settings
- `/iblai-vibe-billing` — add the tenant Billing surface (plan and credits, workspace spend limit, agent limits)
- `/iblai-vibe-memory` — add the tenant Memory admin surface (every user's global memories, every agent's memories)
- `/iblai-vibe-credit` — add the credit balance widget
- `/iblai-vibe-analytics` — add analytics dashboards and reporting views
- `/iblai-vibe-navbar` — add a responsive navbar with logo, links, notification bell, and profile dropdown
- `/iblai-vibe-notification` — add the notification bell and notification center flows
- `/iblai-vibe-invite` — add user invitation dialogs for organization admins
- `/iblai-vibe-workflow` — add workflow builder components (sidebar, modals, connectors)
- `/iblai-vibe-local-llm` — contract for on-device LLM inference (Ollama) in a Tauri desktop build
- `/iblai-vibe-course-access` — add edX course-content pages with outline sidebar, tab strip, iframe, and access control
- `/iblai-vibe-course-create` — drive the ibl.ai Course Creation API to generate, edit, and publish edX courses
- `/iblai-vibe-onboard` — design and build a high-converting questionnaire-style onboarding flow
- `/iblai-vibe-component` — overview of all components and app creation paths
- `/iblai-vibe-scaffold` — scaffold a new app, or add ibl.ai features to an existing Next.js project
- `/iblai-vibe-rbac` — reference for the default RBAC roles, the platform's action-definitions endpoint, and the SDK components that render the Roles + Policies management UI
- `/iblai-vibe-credential` — RBAC setup that lets a token list and unmask integration credentials
- `/iblai-vibe-crm-overview` — reference and index for the Platform-scoped CRM API

#### Agent-tab skills

One skill per tab of the agent management surface: `/iblai-vibe-agent-search` (browse page), `/iblai-vibe-agent-setting`, `/iblai-vibe-agent-access`, `/iblai-vibe-agent-api`, `/iblai-vibe-agent-audit`, `/iblai-vibe-agent-billing`, `/iblai-vibe-agent-dataset`, `/iblai-vibe-agent-disclaimer`, `/iblai-vibe-agent-embed`, `/iblai-vibe-agent-evals`, `/iblai-vibe-agent-grader`, `/iblai-vibe-agent-history`, `/iblai-vibe-agent-llm`, `/iblai-vibe-agent-lti`, `/iblai-vibe-agent-mcp`, `/iblai-vibe-agent-memory`, `/iblai-vibe-agent-privacy`, `/iblai-vibe-agent-prompt`, `/iblai-vibe-agent-safety`, `/iblai-vibe-agent-sandbox`, `/iblai-vibe-agent-skills`, `/iblai-vibe-agent-support`, `/iblai-vibe-agent-task`, `/iblai-vibe-agent-tool`, and `/iblai-vibe-agent-voice`.

#### Monetization skills

`/iblai-vibe-monetization` indexes the family: `/iblai-vibe-monetization-onboard` (Stripe Connect Express onboarding), `/iblai-vibe-monetization-configure` (the admin monetization tab), `/iblai-vibe-monetization-checkout` (paywall modal, access gate, checkout), `/iblai-vibe-monetization-subscription` (the user Purchases tab), `/iblai-vibe-monetization-app-paywall` (gate a whole app on the tenant's own Stripe key), and `/iblai-vibe-monetization-analytics` (revenue dashboards and subscriber lists).

#### Ops skills

- `/iblai-vibe-ops-init` — start a new project and write its CLAUDE.md
- `/iblai-vibe-ops-build` — build and run the app on desktop and mobile (iOS, Android, macOS, Windows), including signed and notarized release builds
- `/iblai-vibe-ops-deploy` — deploy to ibl.ai hosting, your own container, or a static host
- `/iblai-vibe-ops-release` — generate the Makefile and Fastlane config for App Store and Google Play submission
- `/iblai-vibe-windows-msix` — package a Tauri build as a Windows MSIX
- `/iblai-vibe-iconography` — generate every app-icon size for desktop and mobile from one source image
- `/iblai-vibe-ops-test` — validate the app before it is presented to the user
- `/iblai-vibe-ops-upgrade` — upgrade the SDK and the vibe skills to the latest versions
- `/iblai-vibe-readme` — write or refresh the README

#### Design and code quality

- `/iblai-vibe-design` — design, critique, and polish interfaces against the ibl.ai design system
- `/iblai-vibe-deslop` — audit and harden an existing codebase for production readiness

#### Security skills

Eight authorized-use security skills covering reconnaissance, source-code audits (OWASP Top 10), OSINT, disk forensics, incident triage, cloud configuration auditing, dependency vulnerabilities, and prompt-injection testing:

```text
/iblai-vibe-security-recon              /iblai-vibe-security-incident-triage
/iblai-vibe-security-owasp-audit        /iblai-vibe-security-cloud-audit
/iblai-vibe-security-osint-recon        /iblai-vibe-security-dependency-audit
/iblai-vibe-security-disk-forensics     /iblai-vibe-security-prompt-injection
```

#### Marketing skills (companion repo)

The 43 marketing skills (CRO, copywriting, SEO, paid ads, lifecycle, growth) plus 62 platform CLIs and 80 integration guides live in the companion [`iblai/vibe-marketing`](https://github.com/iblai/vibe-marketing) repo, installed side-by-side:

```bash
npx skills add iblai/vibe-marketing
```

## The ibl.ai Backend

`https://api.iblai.app` is the production backend that powers every Vibe app — you do not need to build, host, or maintain any backend services. It provides SSO authentication (OAuth-based login with session management and RBAC), AI agent infrastructure (create, configure, and serve agents with streaming responses, tool use, and RAG), analytics, and per-organization management — each organization gets its own users, agents, branding, and configuration. A free tier is available.

## AI-Assisted Development (MCP Server)

The [@iblai/mcp](https://www.npmjs.com/package/@iblai/mcp) server gives Claude Code deep knowledge of the ibl.ai platform. Add this to your `.mcp.json` at the project root:

```json
{
  "mcpServers": {
    "iblai": {
      "command": "npx",
      "args": ["-y", "@iblai/mcp"]
    }
  }
}
```

This gives your AI assistant access to component props and usage (`get_component_info`), hook signatures (`get_hook_info`), RTK Query endpoint details (`get_api_query_info`), provider setup code (`get_provider_setup`), and page template generation (`create_page_template`).

## Platform Capabilities

| Feature | Web | macOS | Windows/Surface | iOS | Android |
|---------|-----|-------|-----------------|-----|---------|
| SSO Authentication | Yes | Yes | Yes | No | No |
| AI Chat | Yes | Yes | Yes | Yes | Yes |
| User Profile | Yes | Yes | Yes | Yes | Yes |
| Account Settings | Yes | Yes | Yes | Yes | Yes |
| Analytics Dashboard | Yes | Yes | Yes | Yes | Yes |
| Notifications | Yes | Yes | Yes | Yes | Yes |

> **iOS & Android SSO limitation:** Mobile WebViews use a non-standard user-agent that SSO providers reject. Completing the OAuth flow requires a system browser popup (ASWebAuthenticationSession on iOS, Chrome Custom Tabs on Android). This is not yet implemented — mobile users must authenticate via another method for now.

## Quick Start

Install the skills into any project:

```bash
npx skills add iblai/vibe
```

Scaffold a complete app with auth, AI chat, profiles, and more in one command (see the `iblai-app-cli` for the `iblai` CLI installation guide):

```bash
iblai startapp agent -o iblai-init
cp -a iblai-init/<app-name>/. . && rm -rf iblai-init
rm -rf node_modules && pnpm install
cp .env.example .env.local
pnpm dev
```

Open `http://localhost:3000`. You will be redirected to `login.iblai.app` — sign in or create a free account, and you are back in your app with a fully authenticated session.

Already have a project? Install the skills, then use the CLI to add features:

```bash
iblai add mcp            # MCP servers + skills (run first)
iblai add auth           # SSO authentication
iblai add profile        # User profile dropdown
iblai add account        # Account/organization settings
iblai add analytics      # Analytics dashboard
iblai add notification  # Notification bell
```

Deploy to Vercel, or build native desktop and mobile apps:

```bash
iblai deploy vercel
```

```bash
iblai add builds              # Add Tauri support
iblai builds build            # Desktop build for current platform
iblai builds ios init         # iOS project setup
iblai builds ci-workflow --all  # GitHub Actions for all platforms
```

## How It Fits With the Other Repos

Vibe is the build layer of the ibl.ai open-source family: it teaches you (and your AI agent) how to create new applications on the same backend that the sample apps run on, the API skills operate, and the infra CLI deploys.

- **[github.com/iblai/api](https://github.com/iblai/api)** — agent skills + a chat MCP server to headlessly manage and operate the entire ibl.ai platform via its REST API (`npx skills add iblai/api`)
- **[github.com/iblai/os](https://github.com/iblai/os)** — a complete sample application built with Vibe: the open-source AI agent platform running at os.ibl.ai, yours to fork and modify
- **[github.com/iblai/lms](https://github.com/iblai/lms)** — another complete sample application: an open-source skills intelligence platform (courses, competencies, credentials), yours to fork and modify
- **[github.com/iblai/iblai-infra-cli](https://github.com/iblai/iblai-infra-cli)** — how to deploy the platform on your own infrastructure with Terraform + Ansible, given access to the backend images

## Related Documentation

- [Vibe](/developer/applications/vibe) — the Vibe quick-start reference
- [App CLI](/developer/applications/app-cli) — the `iblai-app-cli` scaffolding tool
- [MCP Servers](/developer/applications/mcp) — the ibl.ai MCP server collection
- [Notifications](/developer/applications/notifications) — the notifications system
- [RBAC](/developer/rbac/rbac) — roles, policies, and permissions
