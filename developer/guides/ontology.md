# ibl.ai/ontology

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`README.md`](https://github.com/iblai/ontology/blob/main/README.md). This page is generated — edit it in the repository, not here.

**The open-source ontology layer for AI agents.**

Connect the systems you already run, dissect them into business terms and relationships, certify the result, and publish it as MCP servers — for ibl.ai agents, Claude, Codex, and any other MCP client. One ontology. Every system. Your server, your data, any agent.

[Join](https://ibl.ai/join)
[About](https://ibl.ai)
[Docs](https://ibl.ai/docs/os/overview)
[License: MIT](https://github.com/iblai/ontology/blob/main/LICENSE)
[SOC 2 Type II](https://ibl.ai)
[Next.js](https://nextjs.org)
[MCP](https://modelcontextprotocol.io)

[Install on your server](/developer/guides/ontology-install)
[Run it locally](#quick-start)

Ubuntu 22.04, 24.04 or 26.04 on x86-64 · four blocks to paste · built on your server from this source → [looking after the server](/developer/guides/ontology-operations)

[![Watch the demo](https://github.com/iblai/ontology/blob/main/docs/screenshots/demo-video.png)](https://www.youtube.com/watch?v=pAFl0trxwcM)

**▶︎ [Watch the demo](https://www.youtube.com/watch?v=pAFl0trxwcM)** &nbsp;·&nbsp; _Install it on your own server and connect a database to an agent_

[![Watch the HTTP API demo](https://github.com/iblai/ontology/blob/main/docs/screenshots/demo-video-http-api.png)](https://www.youtube.com/watch?v=gvjs9ZLCvrc)

**▶︎ [Watch the HTTP API demo](https://www.youtube.com/watch?v=gvjs9ZLCvrc)** &nbsp;·&nbsp; _Connect the data source, create an MCP, and connect to an agent in 3 minutes_

[Why ibl.ai/ontology](#why-iblaiontology) · [Every system](#every-system-one-ontology) · [Features](#features) · [How it works](#how-it-works) · [Case studies](#case-studies) · [Screenshots](#screenshots) · [Quick Start](#quick-start) · [Deployment](#deployment) · [Architecture](#architecture) · [FAQ](#faq)

---

**SOC 2 Type II** &nbsp;·&nbsp; Universities, enterprises, and governments run on ibl.ai — [read the case studies →](https://ibl.ai/case-studies)

---

## Why ibl.ai/ontology

| Why                                | What it means                                                                                                                                                                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 🔓 **Your code, your data**        | MIT-licensed and built from this source on your own server. The ontology, the database passwords and the MCP servers stay in your network, and ibl.ai/ontology keeps no database of its own.                                    |
| 🧭 **One ontology, every system**  | SQL connections and HTTP API connections become one set of business terms and relationships: a Student is the same Student in the SIS, the LMS and the CRM.                                                                     |
| ✅ **People certify, agents call** | The engine suggests terms, joins and metrics, and a person certifies them. A certified term becomes a tool with a fixed definition and fixed SQL, so an agent picks the right one and cannot widen it.                          |
| 🤖 **Any agent, over MCP**         | ibl.ai agents, Claude, Codex, Cursor or your own code call the same endpoint, and each caller sees only the toolsets its grant reaches.                                                                                         |
| 🏢 **Enterprise-ready**            | Read-only SQL connections, ibl.ai SSO and RBAC, Microsoft Entra ID, Google and custom OIDC, allowed origins and hosts, API keys limited to toolsets, rate limits, and a log of every call and every refusal.                    |
| 🧩 **Built on proven engines**     | [NVIDIA auto-ontology](https://github.com/NVIDIA/auto-ontology) infers the terms, [Google MCP Toolbox](https://github.com/googleapis/mcp-toolbox) serves the tools, and the ibl.ai platform provides sign-in, roles and agents. |

---

## Every system, one ontology

Most agent integrations are wired one system at a time. ibl.ai/ontology reads the systems you already run into one set of terms, and serves them to every agent.

| System                                 |     | Status                                                                                                |
| -------------------------------------- | --- | ----------------------------------------------------------------------------------------------------- |
| **PostgreSQL**                         | 🐘  | Dissected into terms and served as tools                                                              |
| **MySQL**                              | 🐬  | Dissected into terms and served as tools                                                              |
| **Snowflake**                          | ❄️  | Dissected into terms and served as tools                                                              |
| **ClickHouse**                         | ⚡  | Dissected into terms and served as tools                                                              |
| **HTTP APIs**                          | 🌐  | Any system with an HTTP API, served as named tools from an OpenAPI document or the endpoints you name |
| **Trino · Databricks**                 | 🧱  | Dissected into terms and shown on the map; not served as tools                                        |
| **DuckDB · HeavyDB · Kyuubi · SQLite** | 🔜  | SQL connections coming soon                                                                           |
| **SQL Server · Oracle · BigQuery**     | 🔜  | SQL connections coming soon                                                                           |

**An HTTP API connection reaches everything else.** If a system has an HTTP API, it can be
connected today, with no connector to wait for: a CRM, an SIS, an LMS, a ticketing system, a
SaaS product or a service you wrote yourself. Point at its OpenAPI document and tick the
endpoints, or name them by hand; each becomes a tool. Reads and writes go in separate toolsets,
so you decide per agent whether it may only look or also act, and the upstream credential stays
on your server.

On the other side, anything that speaks MCP can call it: ibl.ai agents, attached in one dialog; Claude Desktop and Claude Code; Codex CLI; Cursor and Windsurf; your own code with the MCP SDK; cURL ([connecting agents](/developer/guides/ontology-connect-agents)).

---

## Features

<table>
<tr>
<td width="50%" valign="top">

**🔌 Connect**

- **SQL connections** — PostgreSQL, MySQL, Snowflake and ClickHouse by connection URL, with a schema scope and a connection test
- **HTTP API connections** — any system with an HTTP API, no connector needed: a base URL and an OpenAPI document, then tick the endpoints to wrap as tools
- **Prebuilt toolsets** — raw SQL per database
- **Secrets on your server** — database passwords and upstream API keys are saved there and nowhere else

**🧠 Dissect & model**

- **Map** — tables under the business terms the engine named, clustered by system
- **Ontology** — object types, link types, metrics, actions and tags, each certified or suggested
- **PII detection** — personal columns are tagged, and their samples masked before any reaches a model
- **Catalog** — every database, schema, table and column the engine read

</td>
<td width="50%" valign="top">

**🚀 Publish & serve**

- **MCP servers** — toolsets generated from certified terms, one endpoint per server
- **Revisions** — each server keeps its revisions, and Roll Back puts the previous one back
- **Playground** — run a tool as yourself and see exactly what your grants return
- **Questions in plain words** — with a model key, `ask_question` answers with certified SQL and returns the SQL it ran
- **Attach to an agent** — an ibl.ai agent gets the server, a key of its own and its toolsets in one dialog

**🔐 Govern & operate**

- **Read-only SQL** — SQL connections use a read-only login, so no agent can change your data through them
- **Sign-in** — ibl.ai credentials, Microsoft Entra ID, Google, or a custom OIDC issuer
- **Roles & access** — one ibl.ai policy per toolset; a toolset that writes is reached only by a grant that names it
- **Origins & hosts** — checked by the gateway before the credential
- **Activity** — every tool call and refusal, with the runtime's log beside it
- **Your server** — built from source and run by systemd, with NVIDIA's engine and Caddy in Docker

</td>
</tr>
</table>

---

## How it works

Five steps, one page each. [docs/console.md](/developer/guides/ontology-console) walks through every page with
screenshots.

| Step            | Page                         | What happens                                                                                                                                |
| --------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **1 · Connect** | Systems                      | Add a SQL connection or an HTTP API connection. SQL connections are read-only, as a security measure. The password is saved on your server. |
| **2 · Dissect** | Map                          | The engine reads the tables and names the business terms above them. Terms that span several systems are called out.                        |
| **3 · Model**   | Ontology                     | Review what was inferred and certify it: object types, their properties, links and metrics.                                                 |
| **4 · Expose**  | MCP Gateway → New MCP Server | Pick the toolsets, how callers sign in, and the allowed origins and hosts. The config file is shown as it will be written.                  |
| **5 · Serve**   | MCP Gateway → the server     | Deploy, attach it to an ibl.ai agent or paste a snippet into Claude, Codex or Cursor, and watch the calls arrive.                           |

---

## Built with

| For                                         | ibl.ai/ontology uses                                                                                                                              |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Inferring terms, joins and metrics          | [NVIDIA auto-ontology](https://github.com/NVIDIA/auto-ontology) 1.2.0, as NVIDIA's own published images, with a Postgres of its own               |
| Serving tools over MCP                      | [Google MCP Toolbox](https://github.com/googleapis/mcp-toolbox) 1.16.0, one process per MCP server on your server's loopback                      |
| Sign-in, roles, API keys and agents         | The ibl.ai platform, through [`@iblai/iblai-js`](https://www.npmjs.com/package/@iblai/iblai-js); the shell matches [os.ibl.ai](https://os.ibl.ai) |
| The pages, the host API and the MCP gateway | Next.js 16 and React 19, in this repository                                                                                                       |

---

## Case studies

Universities, enterprises, and government agencies build on ibl.ai — deploying agents on their own infrastructure, with their own models, at a fraction of the cost of closed platforms.

**[Read the case studies →](https://ibl.ai/case-studies)**

---

## Screenshots

<img src="https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/03-map.png" alt="Map" width="820">

**The Map** — tables under the business terms the engine named, clustered by system

<table>
<tr>
<td width="50%"><img src="https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/02-systems.png" alt="Systems">
Systems — add a SQL connection or an HTTP API connection</td>
<td width="50%"><img src="https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/04-ontology.png" alt="Ontology">
Ontology — review and certify object types, links and metrics</td>
</tr>
<tr>
<td width="50%"><img src="https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/06-mcp-new.png" alt="New MCP Server">
New MCP Server — toolsets, sign-in, allowed origins and the generated config</td>
<td width="50%"><img src="https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/07-server-connect.png" alt="Server · Connect">
Connect — attach an ibl.ai agent, or paste a snippet into Claude, Cursor or your code</td>
</tr>
<tr>
<td width="50%"><img src="https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/07b-server-playground.png" alt="Server · Playground">
Playground — run a tool with your own grants</td>
<td width="50%"><img src="https://raw.githubusercontent.com/iblai/ontology/main/docs/screenshots/08-agents.png" alt="Agents">
Agents — the ibl.ai agents that have a server attached</td>
</tr>
</table>

Every page and tab, with what each control does → [docs/console.md](/developer/guides/ontology-console)

---

## Quick Start

This runs ibl.ai/ontology in **demo mode**: sample data, with no ibl.ai account and no engine.
For the real thing, see [With a real engine](#with-a-real-engine) or [Deployment](#deployment).

```bash
git clone https://github.com/iblai/ontology.git
cd ontology
pnpm install
NEXT_PUBLIC_DEMO=1 pnpm dev   # demo mode
```

**Using [Claude Code](https://claude.ai/claude-code)?** Read [`AGENTS.md`](https://github.com/iblai/ontology/blob/main/AGENTS.md) first: this is Next.js 16, and its conventions differ from earlier releases.

Open `http://localhost:3000`. Demo mode needs no ibl.ai account and no
engine: it uses a fixed session and the sample data in [`lib/mock.ts`](https://github.com/iblai/ontology/blob/main/lib/mock.ts), and the SDK
panels show a notice. Node.js 26 and pnpm 12, as [`.tool-versions`](https://github.com/iblai/ontology/blob/main/.tool-versions) names them;
[mise](https://mise.jdx.dev) brings both with `mise install`.

### With a real engine

On one host, as a developer runs it: the pages and the server half (the host API, the MCP gateway,
Toolbox and their files) on one origin.

1. Start the engine from a checkout of NVIDIA/auto-ontology. Fill in its `.env` as its README
   says, but set `APP_URL=http://localhost:3100` and leave `CONNECTION_STRINGS` empty. The override
   runs the engine on the host's network (a Linux host), with its web app on `127.0.0.1:3100` and
   every service listening on loopback only, so a database on this machine is reached at
   `127.0.0.1` by the engine and by ibl.ai/ontology alike (Docker Compose 2.24.4 or later):

   ```bash
   docker compose -f docker-compose.yml -f <this repo>/deploy/auto-ontology.override.yml up -d --build
   ```

2. Give ibl.ai/ontology the engine's key, one of two ways. Sign in at http://localhost:3100 as
   the engine's admin, open the user menu → **API Tokens** → **New token**, and copy it into
   `AO_API_KEY` (it is shown once; an admin's token is needed for the model export). Or leave
   `AO_API_KEY` out and set `AO_ADMIN_EMAIL`, `AO_ADMIN_PASSWORD_FILE` (a file holding the admin's
   password) and `AO_ORIGIN=http://localhost:3100`: it then signs in as the admin and makes its
   own key on the first call that needs the engine.
3. Start it:

   ```bash
   cp .env.example .env.local    # set ONTOLOGY_PLATFORM, and AO_API_KEY or the admin's sign-in
   pnpm dev                      # http://localhost:3000 → signs in with ibl.ai
   ```

   `ONTOLOGY_PLATFORM` is the key of the ibl.ai platform the engine belongs to. With `AO_URL` set
   and no `ONTOLOGY_PLATFORM` the dev server refuses to start. Any other platform you sign in to
   is answered `403` "This server serves another platform."

One engine serves one platform. To serve several from one host, leave `AO_URL` and
`ONTOLOGY_PLATFORM` unset and write each platform's engine, on another machine, to
`$ONTOLOGY_HOME/platforms/<platform>/engine.json` as `{"url": "...", "key": "..."}`
([docs/network.md](/developer/guides/ontology-network#one-platform-or-any)).

---

## Deployment

ibl.ai/ontology runs on your own server, inside your network. The ibl.ai platform provides
sign-in, roles, API keys and agents; your server keeps the ontology, the database passwords and
the MCP servers. One server serves one platform.

Platforms are hosted on `api.iblai.app` unless yours runs its own ibl.ai deployment, and the
install points there by default. A self-hosted platform is under [Advanced](#advanced).

1. **Install it on a fresh Ubuntu server**

   [docs/install.md](/developer/guides/ontology-install) is the whole install: four blocks to paste. The first one
   names your platform and the name the server answers at, and nothing starts without the
   platform:

   ```bash
   ONTOLOGY_PLATFORM=example-university
   ONTOLOGY_HOST=ontology.example.edu
   ```

   ibl.ai/ontology is built on the server from this repository and run by systemd
   ([`deploy/iblai-ontology.service`](https://github.com/iblai/ontology/blob/main/deploy/iblai-ontology.service)); NVIDIA's engine with its
   database, and Caddy for TLS, run beside it in Docker
   ([`deploy/compose.yaml`](https://github.com/iblai/ontology/blob/main/deploy/compose.yaml)). Nothing of ibl.ai's arrives as a prebuilt image.

2. **Sign in**

   Open `https://<host>/` and sign in as an administrator of that platform. The server serves the
   pages, the host API and the MCP gateway, which agents call at
   `https://<host>/<platform>/<slug>/mcp`.

3. **Look after it**

   Settings, giving the engine a model, upgrades, backups and what to do when something stops are
   in [docs/operations.md](/developer/guides/ontology-operations), which also covers running ibl.ai/ontology as
   ibl.ai's prebuilt image instead (`COMPOSE_PROFILES=console`, with the optional
   `deploy/install.sh`). What talks to what is in [docs/network.md](/developer/guides/ontology-network).

> **Note**: ibl.ai/ontology requires the ibl.ai platform for sign-in, roles, API keys and agents. The platform is not included in this repository — visit [ibl.ai](https://ibl.ai) to get started.

### Troubleshooting

- **A block of the install fails.** Blocks 2, 3 and 4 can be pasted again as they are
  ([docs/install.md](/developer/guides/ontology-install#if-a-step-fails)).
- **Something stops later.** `sudo journalctl -u iblai-ontology -n 30` says why for
  ibl.ai/ontology, and `sudo docker compose logs --tail 30 <service>` in `/opt/iblai-ontology`
  for the rest. What each message means is in
  [docs/operations.md](/developer/guides/ontology-operations#when-something-stops).

---

## Configuration

On a server the settings live in `/opt/iblai-ontology/.env`, which Docker Compose and
ibl.ai/ontology's service both read ([docs/operations.md](/developer/guides/ontology-operations#the-settings)).

| Setting                  | Default                         | What it does                                                                                                                                                       |
| ------------------------ | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ONTOLOGY_PLATFORM`      | None: nothing starts without it | Your platform's key, shown at [login.iblai.app/me](https://login.iblai.app/me). The server serves that platform only; another platform's administrator is refused. |
| `ONTOLOGY_HOST`          | `localhost`                     | The host name. Caddy gets the certificate for it.                                                                                                                  |
| `ONTOLOGY_DM_URL`        | `https://api.iblai.app/dm`      | The ibl.ai platform's address, where every sign-in and key is checked.                                                                                             |
| `OPENAI_API_KEY`         | Empty                           | Turns the semantic features on ([a model](/developer/guides/ontology-operations#a-model)).                                                                                            |
| `OPENAI_API_BASE_URL`    | `https://api.openai.com/v1`     | Where the model calls go: OpenAI, or a gateway that speaks its API.                                                                                                |
| `OPENAI_MODEL`           | `gpt-4.1`                       | The chat model.                                                                                                                                                    |
| `OPENAI_EMBEDDING_MODEL` | `text-embedding-3-large`        | The embedding model.                                                                                                                                               |

For `pnpm dev`, [`.env.example`](https://github.com/iblai/ontology/blob/main/.env.example) lists every setting of the one-host shape with its
comment.

---

## Architecture

One codebase: the Next.js 16 app at the repository root, in the ibl.ai shell (the os.ibl.ai header
and sidebar, Open Sans, `#0058cc`). It has two halves: the pages, and the server half (the host
API under `/api/host/*`, the MCP gateway, one Toolbox process per MCP server, and their files).

```
 admins' browsers ──┐
 ibl.ai agents ─────┼─ 443 ─▶ Caddy (TLS) ─▶ ibl.ai/ontology, 127.0.0.1:3000
 Claude · Codex ────┘                        pages · host API · MCP gateway at
                                             /<platform>/<slug>/mcp
                                                │                  │
                                                ▼                  ▼
                                   NVIDIA auto-ontology     MCP Toolbox
                                   127.0.0.1:3100, with     one process per server,
                                   a Postgres of its own    on loopback
                                                │                  │
                                                ▼                  ▼
                                   your databases,          your databases and
                                   read to build terms      HTTP APIs, as tools

 outbound: ibl.ai (every sign-in and key is checked there) · your data sources ·
           Let's Encrypt · your model's address, if you give it a key
```

The same code runs three ways:

| Shape                  | Setting                                          | Behaviour                                                                               |
| ---------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------- |
| Your server            | `ONTOLOGY_PLATFORM`, with the engine in `AO_URL` | All of it, built from source beside NVIDIA's engine and Caddy; serves that one platform |
| Pages hosted elsewhere | `NEXT_PUBLIC_ONTOLOGY_RELAY=1` at build          | Pages only; every `/api/host/*` call is forwarded to the platform's server              |
| One host run by hand   | None, or `AO_URL` with `ONTOLOGY_PLATFORM`       | Pages and server half as one process: `pnpm dev`, or one host behind a TLS proxy        |

The topology, the firewall rules and the hardening checklist are in
[docs/network.md](/developer/guides/ontology-network), sign-in in [docs/auth.md](/developer/guides/ontology-auth), and where every value
on screen comes from in [docs/data-sources.md](/developer/guides/ontology-data-sources).

### Repository layout

| Path                                                                     | What it is                                                                                                                                                                         |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `app/(app)/*`                                                            | One folder per route: dashboard, systems, map, ontology, mcp, mcp/new, mcp/[slug], agents, roles, access, catalog, sync (Compilation), config                                      |
| `components/shell/`                                                      | Sidebar, header, page primitives, shell state, the OS-style footer dialogs hosting SDK tabs and the prototype walkthrough stepper                                                  |
| `components/graph/ontology-graph.tsx`                                    | Canvas force graph used on the Map                                                                                                                                                 |
| `components/ui/`                                                         | shadcn components (base-nova on Base UI)                                                                                                                                           |
| `lib/iblai/`, `store/`, `app/(auth)/`                                    | ibl.ai SSO: session, SDK store, sign-in callback                                                                                                                                   |
| `lib/host/`, `app/api/host/`                                             | The host layer: checks the caller with ibl.ai, proxies the platform's engine, writes its server configs and runs one Toolbox process per server                                    |
| [`lib/host/engine-key.ts`](https://github.com/iblai/ontology/blob/main/lib/host/engine-key.ts)                       | The server's own engine key: signs in as the engine's admin, makes a key and keeps it in `$ONTOLOGY_HOME/engine-key`                                                               |
| [`lib/host/models.ts`](https://github.com/iblai/ontology/blob/main/lib/host/models.ts), `app/internal/models/v1/`    | The loopback model route the engine calls: swaps in the OpenAI model, adds `OPENAI_API_KEY` and sends the call on                                                                  |
| [`instrumentation.ts`](https://github.com/iblai/ontology/blob/main/instrumentation.ts)                               | The start-up check: a server with `AO_URL` and no `ONTOLOGY_PLATFORM` refuses to start                                                                                             |
| [`lib/host/relay.ts`](https://github.com/iblai/ontology/blob/main/lib/host/relay.ts)                                 | On the pages-only build, forwards each `/api/host/*` call to the server the browser names                                                                                          |
| [`lib/server-address.ts`](https://github.com/iblai/ontology/blob/main/lib/server-address.ts)                         | The `x-ontology-server` header, a server's address `https://<host>/iblai`, and the browser's store of it per platform                                                              |
| `app/[platform]/[slug]/mcp/`                                             | The gateway agents call: origin, credential and toolset checks, then Toolbox                                                                                                       |
| `lib/data.ts`, `lib/ibl-flows.ts`                                        | What each page reads and does; the ibl.ai writes (roles, policies, API keys, agents)                                                                                               |
| [`lib/mock.ts`](https://github.com/iblai/ontology/blob/main/lib/mock.ts)                                             | The fixed data demo mode shows                                                                                                                                                     |
| [`deploy/compose.yaml`](https://github.com/iblai/ontology/blob/main/deploy/compose.yaml)                             | What Docker runs on a server: NVIDIA's engine as its own published images, Postgres, Caddy and the four generated secrets; ibl.ai/ontology's own image under the `console` profile |
| [`deploy/iblai-ontology.service`](https://github.com/iblai/ontology/blob/main/deploy/iblai-ontology.service)         | The systemd unit of ibl.ai/ontology built from source on the server ([docs/install.md](/developer/guides/ontology-install))                                                                           |
| [`deploy/install.sh`](https://github.com/iblai/ontology/blob/main/deploy/install.sh)                                 | Optional: prepares an Ubuntu server and installs ibl.ai/ontology as an image ([docs/operations.md](/developer/guides/ontology-operations#the-console-as-an-image))                                    |
| [`deploy/auto-ontology.override.yml`](https://github.com/iblai/ontology/blob/main/deploy/auto-ontology.override.yml) | For development: runs a local checkout of the engine on the host's network, on loopback, its web app on `3100`; not part of a release                                              |
| [`Dockerfile`](https://github.com/iblai/ontology/blob/main/Dockerfile)                                               | ibl.ai/ontology's image, `public.ecr.aws/ibleducation/iblai-ontology`: Next's standalone build of the whole app with the Toolbox binary; the ibl.ai addresses are build arguments  |
| [`.github/workflows/release.yml`](https://github.com/iblai/ontology/blob/main/.github/workflows/release.yml)         | On every push to `main`: release-it for the version and the changelog, then a draft release with the install files                                                                 |
| [`.env.example`](https://github.com/iblai/ontology/blob/main/.env.example)                                           | Environment settings                                                                                                                                                               |
| [`scripts/screenshots.mjs`](https://github.com/iblai/ontology/blob/main/scripts/screenshots.mjs)                     | Regenerates every screenshot in `docs/screenshots/`                                                                                                                                |
| [`scripts/audit-layout.mjs`](https://github.com/iblai/ontology/blob/main/scripts/audit-layout.mjs)                   | Checks every page at six widths for overflow, overlap and sideways scrolling                                                                                                       |
| [`providers/iblai-providers.tsx`](https://github.com/iblai/ontology/blob/main/providers/iblai-providers.tsx)         | ibl.ai SSO provider chain (see docs/auth.md)                                                                                                                                       |
| [`AGENTS.md`](https://github.com/iblai/ontology/blob/main/AGENTS.md)                                                 | Rules for coding agents, written by `next dev`; keep it committed                                                                                                                  |
| [`docs/`](https://github.com/iblai/ontology/tree/main/docs/)                                                         | The documents listed under [Resources](#resources)                                                                                                                                 |
| [`.claude/launch.json`](https://github.com/iblai/ontology/blob/main/.claude/launch.json)                             | Dev-server definition for the Claude desktop preview                                                                                                                               |

---

## Testing

This project is covered by Vitest unit tests beside the code they test, in [`lib/`](https://github.com/iblai/ontology/tree/main/lib/) and [`deploy/`](https://github.com/iblai/ontology/tree/main/deploy/). **Run them for any change** so nothing regresses:

```bash
pnpm test
```

Other useful commands:

| Command                         | What it does                                                                                                                                          |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm typecheck`                | TypeScript 7 (`tsc --noEmit`)                                                                                                                         |
| `pnpm lint`                     | oxlint, zero warnings                                                                                                                                 |
| `pnpm format:check`             | oxfmt (`pnpm format` rewrites)                                                                                                                        |
| `node scripts/audit-layout.mjs` | Checks every page at 375, 768, 1024, 1280, 1440 and 1920 px for overflow, overlap and sideways scrolling (with `NEXT_PUBLIC_DEMO=1 pnpm dev` running) |
| `node scripts/screenshots.mjs`  | Regenerates every screenshot in `docs/screenshots/` (with `NEXT_PUBLIC_DEMO=1 pnpm dev` running)                                                      |

---

## Advanced

### Self-hosted platform

If you run your own ibl.ai deployment, point the server at it: `ONTOLOGY_DM_URL` in the server's
`.env`, and that deployment's addresses in `.env.local` before the build
([docs/operations.md](/developer/guides/ontology-operations#another-iblai-deployment)). If you need the full
backend, reach out at [ibl.ai/contact](https://ibl.ai/contact) for an enterprise license and
deploy it with [iblai/infra-cli](https://github.com/iblai/infra-cli).

### Pages hosted elsewhere

The pages can also be hosted apart from the server. Built with `NEXT_PUBLIC_ONTOLOGY_RELAY=1`,
ibl.ai/ontology serves pages only and forwards every `/api/host/*` call to the platform's server
at `https://<host>/iblai`, which an administrator connects under **Connect a server**
([docs/operations.md](/developer/guides/ontology-operations#a-console-hosted-elsewhere)).

---

## FAQ

[Getting started](#getting-started) · [Connections](#connections) · [Security and data](#security-and-data) · [Models](#models) · [Agents and access](#agents-and-access) · [Running it](#running-it)

### Getting started

#### 1. Can I try it before installing anything?

Yes. Watch [the demo](https://www.youtube.com/watch?v=pAFl0trxwcM) and
[the HTTP API demo](https://www.youtube.com/watch?v=gvjs9ZLCvrc), or run demo mode from the
[Quick Start](#quick-start): sample data, no account and no engine.

#### 2. Do I need an ibl.ai account?

Yes: an ibl.ai platform and an administrator of it. Sign-in, roles and API keys are checked there.

Your platform's key is shown at [login.iblai.app/me](https://login.iblai.app/me).

#### 3. What does it cost, and what is the licence?

ibl.ai/ontology is MIT-licensed. The ibl.ai backend platform it signs in with is separate
([ibl.ai/contact](https://ibl.ai/contact)).

#### 4. How is this different from pointing an agent at a database MCP server?

An agent gets certified business terms, each a tool with a fixed definition and fixed SQL, and it
reaches only the toolsets its grant names.

### Connections

#### 5. Which systems can it connect?

| Connection | Systems                                                       | Status                                               |
| ---------- | ------------------------------------------------------------- | ---------------------------------------------------- |
| SQL        | PostgreSQL, MySQL, Snowflake, ClickHouse                      | Dissected into terms and served as tools             |
| SQL        | Trino, Databricks                                             | Dissected and shown on the map; not served as tools  |
| SQL        | DuckDB, HeavyDB, Kyuubi, SQLite, SQL Server, Oracle, BigQuery | Coming soon                                          |
| HTTP API   | Any system with an HTTP API                                   | Served as tools today, with no connector to wait for |

More: [connect data sources](/developer/guides/ontology-connect-data-sources).

#### 6. My system is not on the list. Can I still connect it?

Yes, if it has an HTTP API. An HTTP API connection needs no connector: give it the base URL and an
OpenAPI document, or name the endpoints by hand.

More: [connect data sources](/developer/guides/ontology-connect-data-sources),
[the HTTP API demo](https://www.youtube.com/watch?v=gvjs9ZLCvrc).

#### 7. Can an agent change my data?

Not by default. SQL connections are read-only, as a security measure.

With an HTTP API connection you can choose to let an agent modify data: the endpoints that write
go in a toolset of their own, which an agent reaches only when you grant it by name.

More: [permissions](/developer/guides/ontology-permissions).

### Security and data

#### 8. Does my data leave my network?

No. ibl.ai/ontology runs on your server, and the ontology, the passwords and the MCP servers stay
there.

If you give the engine a model key, its model calls go to the address you set, with the samples of
personal columns masked first.

More: [a model](/developer/guides/ontology-operations#a-model), [network](/developer/guides/ontology-network).

#### 9. Where are my database passwords kept?

On your server only: in a file that its service alone can read, and in the engine's own database.

More: [connect data sources](/developer/guides/ontology-connect-data-sources).

#### 10. What is exposed to the internet?

No data is exposed by default. Only ports 80 and 443 are open, through Caddy, and anything
behind them that returns data asks for a sign-in or a key.

NVIDIA auto-ontology engine, the engine's database and every MCP
server's process listen on the server's loopback only.

More: [network](/developer/guides/ontology-network).

#### 11. How is personal data handled?

Columns that look personal are tagged and flagged in the Ontology, and you decide per toolset
which tools may return them.

More: [permissions](/developer/guides/ontology-permissions).

#### 12. Can I limit what each person sees, down to the row?

Yes, three ways:

- by the toolsets a grant names,
- by the certified SQL, which a caller cannot widen,
- by binding a parameter to a claim of the caller's verified sign-in.

More: [permissions](/developer/guides/ontology-permissions).

#### 13. Is there an audit log?

Yes. Every tool call and every refusal is logged on your server and shown under Activity, and the
log can be shipped to your SIEM.

More: [permissions](/developer/guides/ontology-permissions).

### Models

#### 14. Do I need an LLM key?

It is not required. Connecting a system, the map, certifying terms and the tools all work without
one.

An OpenAI API key turns on the semantic features of NVIDIA auto-ontology, such as inferred joins
and questions in plain words: set [`OPENAI_API_KEY`](/developer/guides/ontology-operations#a-model) in the server's
`.env`.

#### 15. Does NVIDIA's engine get my OpenAI key?

No. The engine calls a route on the server's loopback, and ibl.ai/ontology adds your key and sends
the call on.

More: [a model](/developer/guides/ontology-operations#a-model).

#### 16. Can I use a model other than OpenAI's?

Yes: any gateway that speaks OpenAI's API, set in `OPENAI_API_BASE_URL`. The embedding model must
be able to answer in 2048 dimensions.

More: [a model](/developer/guides/ontology-operations#a-model).

### Agents and access

#### 17. Which agents can call it?

ibl.ai agents, Claude Desktop and Claude Code, Codex, Cursor, and anything else that speaks MCP.

More: [connect agents](/developer/guides/ontology-connect-agents).

#### 18. Does it work from browser-based agents?

Yes, once their origin is on the server's Allowed Origins.

More: [connect agents](/developer/guides/ontology-connect-agents).

#### 19. How do callers sign in?

With ibl.ai credentials, Microsoft Entra ID, Google or a custom OIDC issuer, chosen per MCP server.

More: [permissions](/developer/guides/ontology-permissions).

#### 20. Who sees what?

Each caller sees only the tools in the toolsets its grant reaches, so two people can get
different lists from the same address.

More: [permissions](/developer/guides/ontology-permissions).

#### 21. How do I take access away?

Revoke the key under Tokens & Origins on the server's page, or remove the group from the toolset's
policy on ibl.ai.

More: [walkthrough](/developer/guides/ontology-console).

#### 22. Are there rate limits?

Yes: 120 requests or 30 tool calls a minute, per caller.

More: [connect agents](/developer/guides/ontology-connect-agents).

### Running it

#### 23. What does the server need?

Ubuntu 22.04, 24.04 or 26.04 on x86-64 is recommended, 4 GB of memory, 15 GB free, and ports 80
and 443 open.

More: [install](/developer/guides/ontology-install).

#### 24. Why is Docker there if it is built from source?

Only NVIDIA's engine, its Postgres and Caddy run in Docker. ibl.ai/ontology itself is built on
your server from this repository.

More: [install](/developer/guides/ontology-install).

#### 25. Can one server serve several platforms?

No. One server serves one platform.

More: [network](/developer/guides/ontology-network#one-platform-or-any).

#### 26. Can I undo a deploy?

Yes. Each MCP server keeps its revisions, and Roll Back puts the previous one back.

More: [walkthrough](/developer/guides/ontology-console).

#### 27. How do I upgrade, back up or remove it?

See [Day to day](/developer/guides/ontology-operations#day-to-day) in the operations guide.

---

## Contributing

We welcome contributions! Open an issue or a pull request. Before you push, run `pnpm format:check`, `pnpm lint`, `pnpm typecheck` and `pnpm test`. If you'll be working with AI-assisted tooling, read [AGENTS.md](https://github.com/iblai/ontology/blob/main/AGENTS.md) first — it holds the Next.js 16 rules that `next dev` writes for coding agents.

---

## Resources

- [Documentation](https://ibl.ai/docs/os/overview)
- [ibl.ai/ontology](https://ibl.ai/ontology) — the product page
- [Install](/developer/guides/ontology-install) — four blocks to paste on a fresh Ubuntu server
- [Operations](/developer/guides/ontology-operations) — looking after that server: settings, a model, upgrades, backups, the image, failures
- [Walkthrough](/developer/guides/ontology-console) — every page with screenshots, and what each control does
- [Connect data sources](/developer/guides/ontology-connect-data-sources) — SQL and HTTP API connections: read-only logins, scope, PII, what Connect & Dissect does, troubleshooting
- [Permissions](/developer/guides/ontology-permissions) — policies and toolsets, sign-in methods, row-level restriction, PII, writes, origins, audit
- [Connect agents](/developer/guides/ontology-connect-agents) — ibl.ai agents, Claude Desktop and Claude Code, Codex, Cursor, Python, cURL
- [Authentication](/developer/guides/ontology-auth) — ibl.ai SSO: the session as iblai/os implements it, how it is wired here, configuration, demo mode
- [Network](/developer/guides/ontology-network) — topology, inbound, outbound to `ext.ibl.ai` and data sources, what runs where, hardening
- [Data sources](/developer/guides/ontology-data-sources) — where every value on screen comes from, and where each thing is saved
- [ibl.ai/os](https://github.com/iblai/os) — the agent platform whose agents call these servers
- [NVIDIA auto-ontology](https://github.com/NVIDIA/auto-ontology) — the engine that infers terms, joins and metrics
- [Google MCP Toolbox](https://github.com/googleapis/mcp-toolbox) — the runtime that serves the tools

---

## License

MIT License. See [LICENSE](https://github.com/iblai/ontology/blob/main/LICENSE) for details.

**[ibl.ai](https://ibl.ai)** · Your organization's AI, under your control.
