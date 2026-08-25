# iblai-ontology — Architecture

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/architecture.md`](https://github.com/iblai/ontology/blob/main/docs/architecture.md). This page is generated — edit it in the repository, not here.

**Codename:** iblai-ontology
**Status:** Architecture for the on-premise knowledge layer, ported and expanded from the design proposal and aligned with what is implemented in this repository.

This document is the master plan. It explains *why* iblai-ontology exists, the seven components that make it up, the design decisions that hold the whole thing together, and the phased rollout. Each component has its own deep-dive page; links are inline and collected at the end.

---

## Executive Positioning

The prevailing approach in higher-education AI is to extract institutional data through a VPN tunnel into the vendor's cloud. The university gets a "kill switch" on the tunnel, but the data — normalized, projected, queryable — lives on someone else's servers. The vendor controls the compute, the access, and the pipeline.

iblai-ontology is fundamentally different. We deploy a unified knowledge layer **inside the university's network**. No data extraction. No VPN to our cloud. No third-party infrastructure holding institutional data. The university owns the data, the knowledge base, and the infrastructure that runs it. We build the plumbing that makes their existing systems queryable by AI — all under their roof.

The **agent runtime** — where the AI models actually execute — is a separate concern. It can run on ibl.ai's infrastructure, on the university's own compute, or anywhere else. The critical insight is that the knowledge layer (the data, the cache, the permissions) stays on-premise and is exposed via MCP so that any *authorized* agent runtime can securely connect to it.

This is the same positioning we use with the federal government:

> **You will have your data, and we may not even have access to it.**

---

## What It Is, Technically

iblai-ontology makes a university's existing systems — PeopleSoft/Oracle, Ellucian Banner, Canvas, Slate, EAB Navigate, Workday, LDAP/Active Directory — queryable by AI agents over the **Model Context Protocol (MCP)**, without copying institutional data to any vendor cloud.

It is delivered as a **hybrid stack**:

- A lean, self-documenting CLI — the `ontology` command (package `iblai_ontology`, distribution `iblai-ontology`) — built on `typer` + `rich`. The CLI and configuration layer are intentionally Django-free, so `ontology config init` works on a fresh checkout before any backend exists.
- A **Django + Celery backend** for the long-running services: the sync engine, the discovery/provisioning engines, the identity gateway, and the outbound MCP server. The CLI dispatches into the backend (via a `bootstrap()` call) only for commands that need it.

The runtime is deployed as a **Docker Compose** stack on university infrastructure. See [deployment.md](/developer/guides/ontology-deployment).

---

## Architecture Overview

The on-premise knowledge layer has seven components. Components 1–4 form the **data and access layer**. Components 5–7 form the **operations layer** — the tooling that makes the system discoverable, provisionable, and manageable. The agent runtime is separate and connects remotely via MCP.

```
UNIVERSITY NETWORK (ON-PREMISE)
================================================

  Source Systems (existing)
  PeopleSoft  Canvas  Slate  Navigate  Workday  LDAP/AD
      |          |       |       |        |       |
      v          v       v       v        v       v
  +-------------------------------------------------+
  | COMPONENT 1: MCP INBOUND CONNECTIVITY            |
  |                                                   |
  |  MCP Toolbox for        Custom MCP Servers        |
  |  Databases              (Canvas, Slate,           |
  |  (Oracle, Postgres,      Navigate, LDAP)          |
  |   SQL Server)                                     |
  +-------------------------+-------------------------+
                            |
                            v
  +-------------------------------------------------+
  | COMPONENT 2: KNOWLEDGE MATERIALIZATION           |
  |              ("iblai-ontology")                   |
  |                                                   |
  |  Sync Engine (scheduled)                          |
  |       |                                           |
  |       v                                           |
  |  Text Memories    Structured Cache    Vector      |
  |  (Markdown)       (PostgreSQL)        Index       |
  |  /ontology/       students, courses,  (ChromaDB)  |
  |  students/        enrollment, holds,              |
  |  courses/         financial_aid ...               |
  |  financial-aid/                                   |
  +-------------------------------------------------+
                            |
                            v
  +-------------------------------------------------+
  | COMPONENT 3: IDENTITY & PERMISSIONS              |
  |                                                   |
  |  Microsoft Entra ID (Azure AD)                    |
  |  SAML / OIDC / OAuth 2.0                          |
  |                                                   |
  |  JWT token validation on every request            |
  |  Role-to-toolset mapping (roles.yaml)             |
  |  X-Iblai-Role header (set by ibl.ai platform)     |
  +-------------------------------------------------+
                            |
                            v
  +-------------------------------------------------+
  | COMPONENT 4: MCP OUTBOUND EXPOSURE               |
  |                                                   |
  |  iblai-ontology exposed as an MCP server          |
  |  over HTTPS (port 443, streamable_http)           |
  |                                                   |
  |  Firewall-gated: only authorized IPs/networks     |
  |  Token-authenticated: Entra ID OAuth 2.0          |
  |  Toolsets scoped by caller's role                 |
  +-------------------------------------------------+
          |
          | HTTPS (MCP over streamable_http)
          | Through university firewall
          v

EXTERNAL
================================================

  +----------------------------------------------+
  |  AGENT RUNTIME (ibl.ai infrastructure)        |
  |                                                |
  |  Enrollment  Financial  Advising  IT Help      |
  |  Agent       Aid Agent  Agent     Desk Agent   |
  |                                                |
  |  Agents connect to the on-premise ontology     |
  |  via MCP. They forward the user's Entra ID     |
  |  token + X-Iblai-Role on every request. The    |
  |  on-premise layer validates the token and      |
  |  scopes the response to the user's role.       |
  |                                                |
  |  Agents never store institutional data.        |
  +----------------------------------------------+

OPERATIONS LAYER (Components 5-7)
================================================

  +----------------------------------------------+
  |  COMPONENT 7: CLI (ontology)                  |
  |                                                |
  |  $ ontology service add                        |
  |  $ ontology service list / status / sync       |
  |  $ ontology config init / show / set           |
  |  $ ontology sync run / status / history        |
  |  $ ontology health / deploy / mcp / data       |
  +----------------------------------------------+
          |
          v
  +----------------------------------------------+
  |  COMPONENT 5: SERVICE DISCOVERY ENGINE        |
  |                                                |
  |  Connect -> Safety Check -> Schema Introspect |
  |      -> LLM Analysis -> Generate Config       |
  |                                                |
  |  Pre-built adapters: PeopleSoft, Banner,       |
  |  Canvas, Slate, Workday, Navigate, Generic     |
  +----------------------------------------------+
          |
          v
  +----------------------------------------------+
  |  COMPONENT 6: PROVISIONING ENGINE             |
  |                                                |
  |  Generate: cache schema, text templates,       |
  |  tools.yaml, sync-schedules.yaml,             |
  |  docker-compose updates                        |
  |                                                |
  |  Validate: end-to-end test sync per service    |
  +----------------------------------------------+
          |
          v
  [Components 1-4 are configured and running]
```

---

## The Seven Components

| # | Component | One-line summary | Deep dive |
|---|-----------|------------------|-----------|
| 1 | **MCP Inbound Connectivity** | Google MCP Toolbox for databases + custom MCP servers for APIs give every source system a uniform MCP interface. | [components/01-mcp-inbound.md](https://github.com/iblai/ontology/blob/main/docs/components/01-mcp-inbound.md) |
| 2 | **Knowledge Materialization** | The sync engine writes results into Markdown text memories, a PostgreSQL structured cache, and a ChromaDB vector index. | [components/02-knowledge-materialization.md](https://github.com/iblai/ontology/blob/main/docs/components/02-knowledge-materialization.md) |
| 3 | **Identity & Permissions** | Microsoft Entra ID authenticates every request; the gateway validates the JWT and resolves the caller's role against `roles.yaml`. | [components/03-identity.md](https://github.com/iblai/ontology/blob/main/docs/components/03-identity.md) → [identity.md](/developer/guides/ontology-identity) |
| 4 | **MCP Outbound Exposure** | iblai-ontology is itself exposed as an MCP server over `streamable_http`, gated by firewall + Entra tokens, scoped by role. | [components/04-mcp-outbound.md](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md) → [platform-integration.md](/developer/guides/ontology-platform-integration) |
| 5 | **Service Discovery Engine** | Read-only safety suite, schema introspection, BYOK LLM analysis, and pre-built adapters integrate a new source interactively. | [components/05-service-discovery.md](https://github.com/iblai/ontology/blob/main/docs/components/05-service-discovery.md) |
| 6 | **Provisioning Engine** | A 6-step idempotent pipeline turns approved discovery output into a running, cached, queryable integration. | [components/06-provisioning.md](https://github.com/iblai/ontology/blob/main/docs/components/06-provisioning.md) |
| 7 | **CLI (`ontology`)** | A comprehensive `typer`-based command surface for every management operation. | [components/07-cli.md](https://github.com/iblai/ontology/blob/main/docs/components/07-cli.md) |

---

## Key Design Decisions

These principles are non-negotiable and recur throughout every component.

### Read-only everywhere

No source-system writes in v1. Every connection is SELECT-only / GET-only. Before iblai-ontology touches a single row, the **safety verification suite** (Component 5) proves the supplied credentials cannot write — it attempts seven write operations and requires that *all seven be denied*. If any write succeeds, the system refuses to proceed and prints remediation SQL. Write-back, behind human approval workflows, is an explicit future item (see Open Design Questions).

### Credential isolation

Each MCP server has its own credential scope. PeopleSoft credentials are never accessible from the Canvas MCP server. Source-system credentials are encrypted at rest (Fernet) in the service registry and supplied to each container via its own env file.

### No data crosses MCP boundaries

Each MCP server is a process-isolated container. Data flows *up* to the sync engine or the exposure gateway — never *sideways* between MCP servers. On the Docker network, the `ontology-internal` network is marked `internal: true`, so the databases, MCP Toolbox, and sync engine cannot reach the internet at all; only the gateway and the reverse proxy bridge to the outside.

### Identity is borrowed, not built

We use the university's existing Entra ID for authentication. No new passwords, no new MFA. The on-premise gateway validates the JWT and answers exactly one authorization question: *"Given this role, what can the user access?"* It does not decide *who has* a role. See the split below.

### Clean separation of identity, role assignment, and role permissions

| Concern | Where it lives | Managed by |
|---|---|---|
| Who is this user? (identity) | Entra ID JWT | University SSO |
| What role does this user have? (role assignment) | ibl.ai platform (Option A, implemented first) | Platform admin |
| What can this role access? (role permissions) | On-premise `roles.yaml` | Ontology deployer |

### Agent-native storage

Knowledge is materialized primarily as **Markdown text files**, which LLMs read and reason over without ORMs, query languages, or schema mapping. The files are inspectable, git-diffable, and permission-mappable at the filesystem level. A PostgreSQL cache backs the queries that need structure (aggregations, joins, analytics), and a vector index enables semantic search.

---

## Implementation Decisions (as implemented in this repo)

Where the original design proposal left options open, this repository commits to a specific shape. These are stated here as fact; component docs note them again where relevant.

- **Hybrid stack.** A Django-free CLI/config layer (`iblai_ontology`) plus a Django + Celery backend for long-running services. CLI commands that need the backend call `bootstrap()` lazily.
- **Identity Option A first.** The gateway validates the Entra ID JWT and **trusts an `X-Iblai-Role` header** that the ibl.ai platform sets per request; `roles.yaml` maps role → access. Option B (on-prem OAuth, Notion-style, exposing RFC 9728 Protected Resource Metadata + RFC 7591 Dynamic Client Registration) is documented as the roadmap alternative and can coexist. See [identity.md](/developer/guides/ontology-identity).
- **LLM defaults are BYOK.** The university provides its own key; iblai-ontology never ships one. Default models: `anthropic` → `claude-opus-4-8`, `openai` → `gpt-4o`. Without a key, a rule-based analyzer handles common higher-ed schema patterns.
- **Canonical config and SQL.** Large configuration and schema artifacts live in `config/` (`tools.yaml`, `roles.yaml`, `sync-schedules.yaml`, `services.yaml`, `ontology.yaml`) and `sql/` (`schema.sql`, `views.sql`). The docs summarize and reference these rather than reproducing every statement.

---

## Platform Integration (facts from the live ibl.ai codebase)

The ibl.ai platform already models **MCP Server**, **MCP Server Connection**, and **Connected Service**. It implements `discover_and_register_mcp_oauth_service` using **RFC 9728** (Protected Resource Metadata discovery) and **RFC 7591** (Dynamic Client Registration). Per-user in-chat OAuth creates a `ConnectedService` plus an `MCPServerConnection`. The runtime resolution chain is **user → agent → platform** scope, and the user's role is forwarded via `extra_headers` as `X-Iblai-Role`. Registration endpoints live under:

```
base.manager.iblai.app/api/ai-agent/orgs/<org>/users/admin/mcp-servers/
base.manager.iblai.app/api/ai-agent/orgs/<org>/users/admin/mcp-server-connections/
```

Full detail, including the curl calls and the in-chat OAuth sequence, is in [platform-integration.md](/developer/guides/ontology-platform-integration).

---

## Competitive Positioning

| Dimension | Others | iblai-ontology |
|---|---|---|
| **Data location** | Extract to vendor cloud | Stays on the university's own infrastructure |
| **Data control** | "Kill switch" on the pipeline | Full ownership — code, data, and compute are theirs |
| **Vendor data access** | Full access on vendor servers | ibl.ai may have *zero* access — queried at runtime, scoped per user |
| **Compliance story** | "We protect your data" | "You have your data; we may not even have access to it" |
| **Source integration** | Proprietary pipelines | Open standards (MCP) + Google tooling |
| **Air-gap capable** | Requires VPN tunnel | Fully self-contained; can run with zero external network access |
| **Lock-in** | Proprietary platform | Open standards, text files, PostgreSQL, Docker |

---

## Phased Implementation

A condensed view of the rollout; the full week-by-week checklist is in [deployment.md](/developer/guides/ontology-deployment).

- **Phase 1 — Foundation (Weeks 1–3).** Stand up Postgres + MCP Toolbox + ChromaDB, connect the first source (PeopleSoft), run the initial full sync, bring up the gateway + reverse proxy, configure the Entra ID app registration, register iblai-ontology in the ibl.ai platform, and validate an end-to-end agent interaction with role scoping.
- **Phase 2 — Expand sources (Weeks 3–6).** Add Canvas and Slate MCP servers, enrich student memory files, build and test the Financial Aid and Advising agents, and verify role-based access boundaries.
- **Phase 3 — Full iblai-ontology (Weeks 6–10).** Add Navigate, LDAP/AD, and others; build department agents; enable vector search; build the admin dashboard; production-harden (TLS, backups, monitoring, log aggregation); load test.
- **Phase 4 — Advanced (ongoing).** Event-driven sync via webhooks, student-facing self-service agents, cross-institutional templates, and write-back with approval workflows.

---

## The Shared IT Vision

Because iblai-ontology deploys as a standard Docker Compose stack with a standard directory structure and MCP-based connectors, multiple institutions can run the *same* stack independently:

1. Build the template ontology once for PeopleSoft + Canvas + Slate.
2. Deploy at a small institution — same stack, different credentials.
3. Deploy across a consortium — same stack, same pattern.
4. Share agents — the Financial Aid Agent does not need to be rebuilt per school.
5. Stay isolated — each institution runs its own stack; no data sharing unless explicitly configured.

---

## Open Design Questions

1. **Per-user vs. per-entity memory files.** Current design is per-entity (one file per student). A per-user view that aggregates only role-relevant entities trades storage for permission simplicity.
2. **Embedding model for air-gapped deployments.** `nomic-embed-text` runs locally but is less capable; it needs benchmarking against university-scale queries.
3. **Write-back architecture.** When agents eventually update source systems (e.g., approve an aid package), the proposed flow is: agent creates a pending action → human approves in the UI → action executes via an MCP write tool.
4. **Sync conflict / freshness signaling.** Every memory file carries a "Last synced" timestamp; agents should state confidence in data recency when source data may have changed mid-cycle.
5. **Multi-institution sharing.** For the consortium vision, is there a federated layer where institutions can share aggregate, anonymized data for benchmarking with explicit consent?
6. **Multi-tenant MCP server sharing.** The platform supports `is_featured` servers shared across tenants. Should each university register its own MCP Server, or should a featured meta-server route to the correct on-premise instance by tenant?

---

## Document Index

- [components/01-mcp-inbound.md](https://github.com/iblai/ontology/blob/main/docs/components/01-mcp-inbound.md) — Component 1
- [components/02-knowledge-materialization.md](https://github.com/iblai/ontology/blob/main/docs/components/02-knowledge-materialization.md) — Component 2
- [components/03-identity.md](https://github.com/iblai/ontology/blob/main/docs/components/03-identity.md) — Component 3 (pointer)
- [components/04-mcp-outbound.md](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md) — Component 4
- [components/05-service-discovery.md](https://github.com/iblai/ontology/blob/main/docs/components/05-service-discovery.md) — Component 5
- [components/06-provisioning.md](https://github.com/iblai/ontology/blob/main/docs/components/06-provisioning.md) — Component 6
- [components/07-cli.md](https://github.com/iblai/ontology/blob/main/docs/components/07-cli.md) — Component 7
- [identity.md](/developer/guides/ontology-identity) — Component 3 deep dive (Entra ID flow, roles.yaml, Option A vs B)
- [platform-integration.md](/developer/guides/ontology-platform-integration) — Component 4 deep dive (ibl.ai platform)
- [deployment.md](/developer/guides/ontology-deployment) — Docker Compose stack and phased rollout
