# OS (`iblai/os`)

> **GitHub: [github.com/iblai/os](https://github.com/iblai/os)** — The open-source AI agent platform. Build, deploy, and manage intelligent conversational agents — from prototype to production — in minutes. MIT license · Next.js · runs on web, macOS, iOS, Android, Windows, and Linux.

## Overview

OS is the open-source AI agent platform — a complete, production-grade application for building, deploying, and managing conversational AI agents, running live at [os.ibl.ai](https://os.ibl.ai). It is the frontend for the ibl.ai platform: the backend provides authentication, AI agent APIs, and data services, while this repository is the full application codebase you can clone, run, and modify.

In the five-repo family, OS is a **complete sample application built on the ibl.ai backend** — proof of what the stack produces. `iblai/vibe` is the toolkit used to build apps like this one, `iblai/lms` is a second complete sample application (a skills intelligence platform), `iblai/api` operates the platform headlessly via its REST API, and `iblai/iblai-infra-cli` deploys the backend on your own infrastructure.

It is for teams who want a full-featured AI agent product they own end to end: take the codebase, point it at your organization, rebrand it, and ship it — as a web app or as native desktop and mobile apps from the same codebase.

## Features

- **AI Agents** — create custom agents with configurable LLMs, system prompts, tools, and safety filters
- **RAG Training** — upload documents, connect Google Drive, OneDrive, Dropbox, or crawl websites to ground agents in your data
- **Voice Calls** — real-time WebRTC voice chat powered by LiveKit
- **Deep Research** — extended multi-step reasoning for complex queries
- **Canvas / Artifacts** — generate, edit, and version rich documents alongside chat
- **Screen Sharing** — share your screen directly inside a chat session
- **Web Search** — ground responses with live web results
- **MCP Servers** — extend agent capabilities with Model Context Protocol tool servers (GitHub, Notion, Slack, and more)
- **Analytics** — usage dashboards, topic analysis, transcript viewer, and financial reporting
- **Projects** — collaborative workspaces to group agents with shared context and goals
- **Cross-Platform** — ships as web, desktop (macOS, Windows, Linux), and mobile (iOS, Android)
- **Multi-organization** — full isolation with per-organization configuration, branding, and user management
- **SSO** — Single Sign-On with configurable identity providers (OAuth, OIDC, SAML)
- **RBAC** — granular role-based access control with policies and group-based permissions
- **Stripe Billing** — subscription management, free trials, and usage-based pricing
- **Embed Mode** — embed agents in any website via iframe with custom styling
- **Custom Domains** — host agents on your own domain
- **API Keys** — programmatic access for integrations and automation
- **Whitelabeling** — custom branding, logos, and disclaimers

## Available On

| Platform | Status |
|----------|--------|
| **Web** | Production at [os.ibl.ai](https://os.ibl.ai) — works on any modern browser |
| **macOS** | Native desktop app — lightweight, fast, system-integrated |
| **iOS** | Native mobile app — available on iPhone and iPad |
| **Android** | Native mobile app — available on phones and tablets |
| **Windows** | Native desktop app |
| **Linux** | Native desktop app |

One codebase, six platforms. OS runs natively everywhere your users are — lightweight, fast, with near-native performance.

## Deployment

OS is the frontend for the ibl.ai platform, so the path depends on your backend setup. The backend is not included in this repository — visit [ibl.ai](https://ibl.ai) to get started.

#### Option A: Existing ibl.ai organization

If you already have an ibl.ai organization key, configure it and deploy with Docker (recommended):

```bash
cp .env.example .env.local
```

```bash
docker build -t os .
docker run -p 5000:5000 --env-file .env.local os
```

Or deploy standalone — the build emits a self-contained server under `.next/standalone/` (Next.js standalone output):

```bash
pnpm build
PORT=3000 node server-wrapper.js
```

#### Option B: Enterprise deployment

If you need full backend infrastructure, reach out at [ibl.ai/contact](https://ibl.ai/contact) for an enterprise license (full backend codebase). If you already have access to the Docker images, deploy them via [iblai/iblai-infra-cli](https://github.com/iblai/iblai-infra-cli).

#### Desktop & mobile

Native app build instructions live in the repo's `docs/development.md`; full deployment docs are in `docs/standalone-deployment.md`.

## Testing

The project is covered by Playwright end-to-end tests in `e2e/`. Run the suite for any change so nothing regresses:

```bash
make e2e-ui
```

The first time, install the browsers once:

```bash
make e2e-install
```

Other targets: `make e2e` (full headless suite), `make e2e-headed`, `make e2e-chrome`, `make e2e-journey J=01` (single journey), and `make e2e-report`. Coverage is tracked in `e2e/COVERAGE.md` and must not regress.

## Quick Start

```bash
git clone https://github.com/iblai/os.git
cd os
pnpm install
```

Using Claude Code? Run `/setup` — it walks you through connecting your ibl.ai organization and configuring `.env.local` automatically. Manual setup: copy `.env.example` to `.env.local`, then set `NEXT_PUBLIC_MAIN_TENANT_KEY` to your org key from [login.iblai.app/me](https://login.iblai.app/me).

```bash
cp .env.example .env.local   # then edit NEXT_PUBLIC_MAIN_TENANT_KEY
pnpm dev
```

Open `http://localhost:3000`. The repo's `docs/development.md` covers environment variables, scripts, and architecture details, and the README's Troubleshooting section covers the common blank-page causes (a stale server holding the port, or a stray parent lockfile nesting the standalone output).

## How It Fits With the Other Repos

OS is one of the two complete sample applications in the ibl.ai open-source family — a working product you can take and modify, built on the same backend the other repos build against, operate, and deploy.

- **[github.com/iblai/api](https://github.com/iblai/api)** — agent skills + a chat MCP server to headlessly manage and operate the entire ibl.ai platform via its REST API (`npx skills add iblai/api`)
- **[github.com/iblai/vibe](https://github.com/iblai/vibe)** — how to "vibe code" new applications on top of the ibl.ai backend (Next.js SDK + Claude Code skills; `npx skills add iblai/vibe`)
- **[github.com/iblai/lms](https://github.com/iblai/lms)** — the other complete sample application: an open-source skills intelligence platform (courses, competencies, credentials), yours to fork and modify
- **[github.com/iblai/iblai-infra-cli](https://github.com/iblai/iblai-infra-cli)** — how to deploy the platform on your own infrastructure with Terraform + Ansible, given access to the backend images

## Related Documentation

- [Agent Quickstart](/developer/agents/quickstart) — create an agent and chat with it programmatically
- [Standard Agents](/developer/agents/standard) — agent types and configuration
- [MCP Connections](/developer/agents/mcp-authentication/mcp-connections) — connecting MCP servers to agents
- [Vibe](/developer/applications/vibe) — the toolkit OS is built with
- [RBAC](/developer/rbac/rbac) — roles, policies, and permissions
- [Infrastructure CLI](/developer/infrastructure/infra-cli) — provisioning the backend OS runs on
