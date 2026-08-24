# API (`iblai/api`)

> **GitHub: [github.com/iblai/api](https://github.com/iblai/api)** — Operate any ibl.ai organization from your AI agent. Skills + a chat MCP server. 33 skills · MIT license · works with Claude Code, Cursor, GitHub Copilot, and 15+ other skills-compatible agents.

## Overview

`iblai/api` is a toolkit for operating the ibl.ai platform **headlessly** — from a coding agent, a terminal, CI, or any MCP-capable assistant, with no UI required. It packages every agent-configuration and platform-admin operation as a skill that maps directly to its exact REST endpoints on `api.iblai.app` (method, URL, body), plus a hosted MCP server for the one runtime capability that isn't a REST call: chatting with a deployed agent.

In the five-repo family, this is the **management layer**: `iblai/vibe` teaches your agent to *build* new apps on the ibl.ai backend, `iblai/os` and `iblai/lms` are complete sample applications on that backend, and `iblai/iblai-infra-cli` deploys the backend on your own infrastructure — while `iblai/api` lets an AI agent *run the platform itself* via the REST API.

It is for developers and platform administrators who want to configure agents, manage datasets and memory, administer users and roles, send notifications, and pull analytics as `/` commands instead of docs hunts. The skills run against the hosted `api.iblai.app` environment; for a license to the full platform codebase to run locally or self-host, contact [ibl.ai/contact](https://ibl.ai/contact).

## How It Works

1. **Install** — `npx skills add iblai/api` drops the skills into your project.
2. **Connect** — run `/iblai-api-login` to capture your org + username from `login.iblai.app/me` and store an Api-Token in `.env`.
3. **Operate** — invoke any `/iblai-*` skill; it fills in your org, username, and (where relevant) the agent id, then calls `api.iblai.app`.
4. **Automate** — chain skills, or connect the hosted chat MCP server to your assistant to talk to agents at runtime.

Every skill is **endpoint-accurate**: it carries the real `api.iblai.app` request shapes, so your agent calls the platform correctly the first time. Authenticate once and you can target any organization you belong to by org key + Api-Token.

## Skills

After installing, use these directly in your AI agent as `/` commands. One skill per operation — each `/iblai-*` skill maps one capability.

#### Setup

```text
/iblai-api-login
```

Connects an organization — opens `login.iblai.app/me`, captures org + username + Api-Token into `.env`. Run this first.

#### Agent skills

Everything about a single agent — creation, identity, model, prompts, knowledge, safety, and operations:

```text
/iblai-api-agent-create      /iblai-api-agent-dataset
/iblai-api-agent-setting     /iblai-api-agent-embed
/iblai-api-agent-sandbox     /iblai-api-agent-memory
/iblai-api-agent-access      /iblai-api-agent-history
/iblai-api-agent-llm         /iblai-api-agent-audit
/iblai-api-agent-prompt      /iblai-api-agent-eval
/iblai-api-agent-skill       /iblai-api-agent-chat
/iblai-api-agent-safety      /iblai-api-agent-disclaimer
/iblai-api-agent-privacy     /iblai-api-agent-tool
/iblai-api-agent-mcp         /iblai-api-agent-session
/iblai-api-agent-support
```

Highlights: `/iblai-api-agent-create` creates an agent from a template; `/iblai-api-agent-llm` picks the LLM provider and model; `/iblai-api-agent-dataset` manages RAG training data (files, URLs, YouTube, crawl, GitHub); `/iblai-api-agent-memory` manages agent memories; `/iblai-api-agent-safety` and `/iblai-api-agent-privacy` cover moderation and PII redaction; `/iblai-api-agent-eval` runs evaluations with LLM-as-Judge and human scoring; `/iblai-api-agent-support` triages and answers human-support tickets; `/iblai-api-agent-session` works with chat sessions; `/iblai-api-agent-mcp` wires up MCP connectors with OAuth.

#### Organization skills (platform admin)

Org-wide administration:

```text
/iblai-api-org               /iblai-api-rbac
/iblai-api-management        /iblai-api-crm
/iblai-api-integration       /iblai-api-notification
/iblai-api-token             /iblai-api-invite
/iblai-api-scim              /iblai-api-billing
/iblai-api-feature           /iblai-api-credential
/iblai-api-infrastructure    /iblai-api-ecosystem
```

Highlights: `/iblai-api-management` administers users, groups, roles, policies, teams, and alerts; `/iblai-api-rbac` covers roles, policies, and permission checks; `/iblai-api-scim` provides SCIM 2.0 directory provisioning; `/iblai-api-billing` handles credits, paywalls, checkout, and subscriptions; `/iblai-api-token` rotates Platform API Tokens; `/iblai-api-credential` lists and unmasks integration credentials.

#### Profile skills

```text
/iblai-api-profile           /iblai-api-profile-metadata
```

The signed-in user's own profile (Basic, Social, Education, Experience, Resume, Memory) and a per-user, per-org metadata key-value store.

#### Content & discovery skills

```text
/iblai-api-search            /iblai-api-course-create
/iblai-api-analytics         /iblai-api-catalog
/iblai-api-milestone         /iblai-api-catalog-media
/iblai-api-apply             /iblai-api-catalog-invitation
```

Highlights: `/iblai-api-search` provides faceted discovery of agents and content plus personalized (RAG) recommendations; `/iblai-api-analytics` spans KPIs, users, topics, transcripts, costs, courses, and reports; `/iblai-api-course-create` drives the Course Creation API; `/iblai-api-catalog` manages courses, programs, pathways, and the skills/roles taxonomy; `/iblai-api-apply` covers application workflows.

#### Inference and proxy skills

```text
/iblai-api-inference         /iblai-api-external-service-proxy
```

`/iblai-api-inference` calls models through the platform; `/iblai-api-external-service-proxy` covers the proxied third-party services documented under [External Service Proxy](/developer/externalserviceproxy/overview).

Skills live in the repo's `skills/` directory — read them, extend them, or write your own.

## MCP Server

A hosted Model Context Protocol server — no local installation required — covers the one **runtime** capability the skills can't: actually talking to a deployed agent, with streamed responses, tool use, and RAG. Wire it up with `/iblai-api-agent-chat` (it writes the config from your `.env` token + a chosen agent), or add it manually.

#### Connect from Claude Code

```bash
claude mcp add iblai-agent-chat --transport http https://asgi.data.iblai.app/mcp/agent-chat/ --header "Authorization: Api-Token YOUR_API_TOKEN"
```

#### Connect from Claude Desktop / Cursor

```json
{
  "mcpServers": {
    "iblai-agent-chat": {
      "transport": "streamable-http",
      "url": "https://asgi.data.iblai.app/mcp/agent-chat/",
      "headers": {
        "Authorization": "Api-Token YOUR_API_TOKEN"
      }
    }
  }
}
```

#### Skills vs the MCP server

The split is **administer/use-via-REST vs chat at runtime**. Skills do everything reachable over REST — creating and configuring agents, datasets, memory, users, roles, notifications, discovery, profiles, analytics, and reporting. The MCP server exists only for holding a live conversation with a deployed agent; if a skill covers an operation, there is no server for it.

## Authentication

Everything authenticates the same way:

- **Base URL:** `https://api.iblai.app`
- **Header:** `Authorization: Api-Token <key>` on every request
- **Org & username:** from [login.iblai.app/me](https://login.iblai.app/me) — each organization you belong to shows its key (e.g. `enterprise`, `iblai`, or a UUID)
- **Api-Token:** `/iblai-api-login` mints your first token from your signed-in session; afterward `/iblai-api-token` lists, creates, and rotates tokens. The secret is shown once.

Never commit `.env` — it is in `.gitignore`.

## Quick Start

You need [Node.js](https://nodejs.org) (for `npx`), a skills-compatible AI agent (Claude Code, Cursor, OpenCode, and 15+ others), and an ibl.ai account. No account yet? Sign up at [ibl.ai/join](https://ibl.ai/join) — it creates your account and your organization.

Install the skills:

```bash
npx skills add iblai/api
```

Then connect your organization (it writes `IBLAI_ORG`, `IBLAI_USERNAME`, and `IBLAI_API_KEY` to `.env`):

```text
/iblai-api-login
```

Every other skill then reads those values from `.env` and calls `https://api.iblai.app` with `Authorization: Api-Token <key>`.

## How It Fits With the Other Repos

`iblai/api` is the headless management layer of the ibl.ai open-source family: it operates the same backend that the sample applications run on and that the infra CLI deploys.

- **[github.com/iblai/vibe](https://github.com/iblai/vibe)** — how to "vibe code" new applications on top of the ibl.ai backend (Next.js SDK + Claude Code skills; `npx skills add iblai/vibe`)
- **[github.com/iblai/os](https://github.com/iblai/os)** — a complete sample application on the backend: the open-source AI agent platform running at os.ibl.ai, yours to fork and modify
- **[github.com/iblai/lms](https://github.com/iblai/lms)** — another complete sample application: an open-source skills intelligence platform (courses, competencies, credentials), yours to fork and modify
- **[github.com/iblai/iblai-infra-cli](https://github.com/iblai/iblai-infra-cli)** — how to deploy the platform on your own infrastructure with Terraform + Ansible, given access to the backend images

## Related Documentation

- [Agent Quickstart](/developer/agents/quickstart) — create an agent and chat with it programmatically
- [Standard Agents](/developer/agents/standard) — agent types and configuration
- [Agent Memory](/developer/agents/memory) — how agent memory works
- [Agent Evaluations](/developer/agents/evaluations) — evaluation datasets and scoring
- [MCP Connections](/developer/agents/mcp-authentication/mcp-connections) — connecting MCP servers to agents
- [RBAC](/developer/rbac/rbac) — roles, policies, and permissions
- [MCP Servers](/developer/applications/mcp) — the ibl.ai MCP server collection
