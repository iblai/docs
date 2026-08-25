# iblai/ontology

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`README.md`](https://github.com/iblai/ontology/blob/main/README.md). This page is generated — edit it in the repository, not here.

On-premise knowledge layer that makes an organization's existing systems queryable by AI agents over MCP — no data extraction.

  [MCP Toolbox](https://github.com/googleapis/mcp-toolbox)

---

## What it is

iblai/ontology is a **unified knowledge layer that runs inside your network**. It makes the systems you already run — SIS/ERP databases (PeopleSoft, Oracle, Postgres), data warehouses (Snowflake), and SaaS apps (Canvas, Salesforce, ServiceNow, Workday, Jira, …) — queryable by AI agents over the **Model Context Protocol (MCP)**.

The prevailing approach extracts your data through a VPN tunnel into a vendor cloud. iblai/ontology does the opposite: **no data extraction, no VPN to our cloud, no third-party infrastructure holding your data.** The agent runtime (where models execute) is a separate concern and can run anywhere; the knowledge layer — the data, the cache, the permissions — stays on-premise and is exposed over MCP so any *authorized* runtime can connect.

> **You will have your data, and we may not even have access to it.**

It is **domain-agnostic**: the same stack serves a university (higher-ed systems) and an enterprise (CRM/ITSM/HCM/data-warehouse), with built-in defaults for both (see [the catalog](#built-in-service-catalog)).

## How it connects (MCP in, MCP out)

```
   YOUR NETWORK (ON-PREMISE)
   ┌──────────────────────────────────────────────────────────────┐
   │  Source systems                                                │
   │  PeopleSoft/Oracle · Snowflake · Postgres   Canvas · Salesforce │
   │        │  (SQL, read-only)                   │ (REST)           │
   │        ▼                                      ▼                  │
   │  ┌───────────────────────────┐   ┌──────────────────────────┐  │
   │  │ Google MCP Toolbox        │   │ Custom MCP servers        │  │
   │  │ (databases) [1]           │   │ (APIs) [1]                │  │
   │  └─────────────┬─────────────┘   └────────────┬─────────────┘  │
   │                └───────────────┬───────────────┘                │
   │            [2] sync → text memories (MD) + Postgres cache + vectors │
   │                                │                                │
   │            [3] identity: Entra ID JWT + roles.yaml (X-Iblai-Role) │
   │                                │                                │
   │            [4] iblai/ontology exposed as ONE MCP server (HTTPS) │
   └────────────────────────────────┬───────────────────────────────┘
                                     │  MCP over HTTPS, role-scoped
                                     ▼
   EXTERNAL: agent runtime (ibl.ai or anywhere) — forwards the user's
   Entra token per request, never stores institutional data.
```

- **Inbound** uses [**Google MCP Toolbox for Databases**](https://github.com/googleapis/mcp-toolbox): databases are exposed as MCP tools via `config/tools.yaml` (`kind: source | tool | toolset`). REST systems get lightweight custom MCP servers (`mcp-servers/`). Run `ontology mcp validate` to check `tools.yaml` against the Toolbox schema.
- **Outbound**, iblai/ontology is itself **one MCP server** (`streamable_http`, behind your firewall + Entra ID). Any MCP client — the ibl.ai agent runtime, Claude, Cursor, a custom app — connects and gets results scoped to the caller's role.

Full design: **[docs/architecture.md](/developer/guides/ontology-architecture)**.

## Quick start

```bash
pip install -e ".[dev]"      # core CLI + tests
ontology --help
ontology config init         # scaffold a deployment (config/, sql/, compose)
```

Hybrid stack — a lean, Django-free CLI + config layer (works on a fresh checkout) and a Django + Celery backend for the long-running services. Install extras as needed:

```bash
pip install -e ".[django]"   # backend services (gateway, sync, discovery)
pip install -e ".[llm]"      # BYOK schema analysis (anthropic / openai)
pip install -e ".[db]"       # source drivers (oracle, postgres, mysql, mssql)
pip install -e ".[vector]"   # ChromaDB vector index
```

Names: distribution **`iblai-ontology`** · import package **`iblai_ontology`** · command **`ontology`**.

### Local checks

`./dev.sh` mirrors the CI gates so you can confirm everything passes before committing:

```bash
./dev.sh setup            # install into the active venv (dev + django extras)
./dev.sh test -k canvas   # pytest (args passed through)
./dev.sh fmt              # auto-fix ruff format + import sort
./dev.sh check            # ruff format/lint/import-sort + full pytest (== CI)
```

CI runs the same on every PR to `main`: `.github/workflows/ruff-format.yml` (ruff) and `.github/workflows/tests.yml` (pytest).

## End to end

### 1 · Discover a service — two ways

**A. Live database** — connect, verify read-only, introspect, analyze, provision:

```bash
ontology service add --from peoplesoft \
  --host psft-db.internal.edu --database CSPRD --user iblai_readonly
```

`--from <catalog-key>` prefills the connection shape (driver, port, env) from the [built-in catalog](#built-in-service-catalog). The pipeline runs a **read-only safety suite first** — seven write attempts that must all be denied — then introspects the schema and generates config.

**B. From a SKILL.md** — seed an API source with no live connection:

```bash
ontology service add --skill canvas      # or: ontology skill import canvas
ontology skill list                      # all vendored skills (higher-ed + enterprise)
```

This parses the skill's connection env and key operations into a discovery seed (read-only operations become suggested MCP tools).

### 2 · Interpret the schema & see the connection

```bash
ontology service schema peoplesoft        # discovered tables, by row count
ontology service connection peoplesoft     # stored connection (secrets redacted)
```

```text
$ ontology service schema peoplesoft
peoplesoft — oracle | 847 tables, 12,400,000 rows
            Top 20 tables
┏━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━┳━━━━━━━━━┓
┃ Schema ┃ Table              ┃ Rows      ┃ Columns ┃
┡━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━╇━━━━━━━━━┩
│ SYSADM │ PS_STDNT_CAR_TERM  │ 2,345,678 │ 24      │
│ SYSADM │ PS_STDNT_ENRL      │ 1,876,543 │ 31      │
└────────┴────────────────────┴───────────┴─────────┘
```

### 3 · Test it

```bash
ontology service test peoplesoft           # 7-test read-only safety suite
ontology mcp validate                       # tools.yaml is MCP Toolbox compliant
ontology mcp test get-student-enrollment --params '{"student_id":"001234567"}'
```

```text
$ ontology service test peoplesoft
  [PASS] CREATE TABLE blocked   [PASS] INSERT blocked   [PASS] UPDATE blocked
  [PASS] DELETE blocked   [PASS] DROP TABLE blocked   [PASS] ALTER blocked
  [PASS] TRUNCATE blocked
  All safety checks passed.   # credentials confirmed read-only
```

### 4 · Provision, sync, serve

```bash
ontology service approve peoplesoft         # cache schema, tools, sync schedules
ontology sync run peoplesoft                # pull → cache + text memories + vectors
ontology deploy up                          # bring up the stack (compose)
```

Then register iblai/ontology as an MCP server in your agent platform (see [docs/platform-integration.md](/developer/guides/ontology-platform-integration)):

```bash
ontology platform register --url https://ontology.your-org.edu/mcp
ontology platform connect  --server 14 --scope user --role FinancialAidCounselor
```

## Built-in service catalog

`ontology catalog list` ships defaults (connection shape, adapter, default toolset, sync cadences, and the upstream SKILL.md) for 22 systems across two domains. Seed any of them with `ontology service add --from <key>` or `ontology skill import <key>`.

**Higher-ed** — [iblai/higher-education-agents](https://github.com/iblai/higher-education-agents)

| Key | System | Type | Skill |
|---|---|---|---|
| `peoplesoft` | PeopleSoft (Oracle) | database | — |
| `banner` | Ellucian Banner | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/banner/SKILL.md) |
| `canvas` | Instructure Canvas LMS | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/canvas/SKILL.md) |
| `slate` | Technolutions Slate CRM | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/slate/SKILL.md) |
| `workday` | Workday HCM / Student | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/workday/SKILL.md) |
| `eab-navigate` | EAB Navigate | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/eab-navigate/SKILL.md) |
| `salesforce-education-cloud` | Salesforce Education Cloud | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/salesforce-education-cloud/SKILL.md) |
| `servicenow` | ServiceNow ITSM | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/servicenow/SKILL.md) |
| `civitas-learning` | Civitas Learning | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/civitas-learning/SKILL.md) |
| `handshake` | Handshake Careers | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/handshake/SKILL.md) |
| `blackbaud-raisers-edge` | Blackbaud Raiser's Edge NXT | api | [SKILL.md](https://github.com/iblai/higher-education-agents/blob/main/skills/blackbaud-raisers-edge/SKILL.md) |

**Enterprise** — [iblai/enterprise-agents](https://github.com/iblai/enterprise-agents)

| Key | System | Type | Skill |
|---|---|---|---|
| `snowflake` | Snowflake Data Warehouse | database | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/snowflake/SKILL.md) |
| `salesforce` | Salesforce CRM / Sales Cloud | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/salesforce/SKILL.md) |
| `hubspot` | HubSpot CRM | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/hubspot/SKILL.md) |
| `servicenow-itsm` | ServiceNow ITSM | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/servicenow/SKILL.md) |
| `jira` | Jira | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/jira/SKILL.md) |
| `confluence` | Confluence | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/confluence/SKILL.md) |
| `github` | GitHub | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/github/SKILL.md) |
| `okta` | Okta Identity | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/okta/SKILL.md) |
| `slack` | Slack | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/slack/SKILL.md) |
| `zendesk` | Zendesk | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/zendesk/SKILL.md) |
| `zoom` | Zoom | api | [SKILL.md](https://github.com/iblai/enterprise-agents/blob/main/skills/zoom/SKILL.md) |

```bash
ontology catalog show snowflake     # connection shape, env, default toolset, cadences, skill
```

## Utilities

```bash
ontology doctor          # diagnostics: config validity, drivers, per-service env, Entra
ontology health          # PostgreSQL cache, MCP servers, sync engine, text-memory storage
ontology catalog list    # built-in service defaults (--domain higher-ed|enterprise)
ontology mcp validate    # tools.yaml ↔ Google MCP Toolbox schema
```

## CLI reference

| Group | Commands | Purpose |
|---|---|---|
| `service` | `add` · `list` · `status` · `test` · `schema` · `connection` · `discover` · `approve` · `sync` · `remove` | Source integrations (discovery + provisioning) |
| `skill` | `list` · `import` | Inspect / seed discovery from a SKILL.md |
| `catalog` | `list` · `show` | Browse built-in service defaults |
| `config` | `init` · `show` · `set` · `llm` · `validate` | Configuration management |
| `sync` | `run` · `status` · `history` · `schedule` | Sync operations |
| `roles` | `list` · `show` · `validate` | Role & permission management |
| `mcp` | `status` · `tools` · `toolsets` · `validate` · `test` | MCP server administration |
| `platform` | `register` · `connect` · `attach` | Register with the ibl.ai platform |
| `health` / `doctor` | (sub-checks) | Diagnostics |
| `deploy` | `up` · `down` · `logs` · `restart` · `status` | Docker Compose lifecycle |

Full reference: **[docs/components/07-cli.md](https://github.com/iblai/ontology/blob/main/docs/components/07-cli.md)**.

## Security posture

- **Read-only everywhere.** No source writes in v1. Before touching any data, the safety suite attempts seven write operations and requires **all seven be denied** — otherwise it refuses to proceed and prints remediation SQL.
- **Data stays on-premise.** Source credentials never leave your network; the agent runtime queries at runtime, scoped by the authenticated user.
- **Identity through Entra ID.** Every MCP request carries the user's Entra ID JWT; the gateway validates it and resolves the caller's role against `roles.yaml`. See [docs/identity.md](/developer/guides/ontology-identity).
- **Credential isolation & containment.** Each inbound MCP server has its own credential scope; connection secrets are encrypted at rest.

### Gateway hardening (environment variables)

The gateway middleware is tuned entirely through environment variables on the
`ontology-gateway` container — no code change or rebuild, just set and restart.

**Rate limiting** (fixed-window throttle, keyed on the authenticated subject and
falling back to client IP):

| Env var | Default | Purpose |
|---|---|---|
| `ONTOLOGY_RATELIMIT_ENABLED` | `true` | Master on/off switch |
| `ONTOLOGY_RATELIMIT_WINDOW` | `60` | Window length in seconds (also the `Retry-After` value) |
| `ONTOLOGY_RATELIMIT_MAX` | `120` | Max requests per window (general bucket) |
| `ONTOLOGY_RATELIMIT_TOOLS_CALL_MAX` | `30` | Stricter max per window for `tools/call` |

To loosen limits for higher traffic, raise `ONTOLOGY_RATELIMIT_MAX` /
`ONTOLOGY_RATELIMIT_TOOLS_CALL_MAX`. Two caveats:
- The window is **fixed**, not sliding — a client can burst up to `2×MAX` across a
  window boundary. Size the limit accordingly.
- Counting is **per worker process** on the default in-memory cache, so with *N*
  workers the effective limit is ≈ `N × MAX`. Set `ONTOLOGY_CACHE_URL` to a
  `redis://…` URL so all workers share one counter and the configured number is
  the true global limit.

**Transport security & response headers:**

| Env var | Default | Purpose |
|---|---|---|
| `ONTOLOGY_SECURITY_HEADERS_ENABLED` | `true` | Emit security response headers |
| `ONTOLOGY_REQUIRE_HTTPS` | `true` | Reject Bearer tokens received over a plaintext (non-HTTPS) connection |
| `ONTOLOGY_HSTS_MAX_AGE` | `31536000` | `Strict-Transport-Security` max-age (seconds; `0` disables HSTS) |
| `ONTOLOGY_HSTS_INCLUDE_SUBDOMAINS` | `true` | Add `includeSubDomains` to HSTS |
| `ONTOLOGY_CSP` | `default-src 'none'; frame-ancestors 'none'` | `Content-Security-Policy` value |
| `ONTOLOGY_REFERRER_POLICY` | `no-referrer` | `Referrer-Policy` value |
| `ONTOLOGY_FRAME_OPTIONS` | `DENY` | `X-Frame-Options` value |

HSTS is only emitted over HTTPS. TLS terminates at the Caddy edge, so the gateway
trusts the proxy's `X-Forwarded-Proto` (`SECURE_PROXY_SSL_HEADER`) to decide
whether a connection is secure — keep the gateway reachable only via the proxy,
never directly on `:8080`.

**JWT replay protection:**

The token `jti` (a required, validated claim) is checked against a cache-backed
replay store keyed on the `jti`, with each entry expiring exactly when the token
does (`exp - now`).

| Env var | Default | Purpose |
|---|---|---|
| `ONTOLOGY_JWT_REPLAY_MODE` | `bind` | `bind` \| `strict` \| `off` (see below) |
| `ONTOLOGY_JWT_REPLAY_TTL_FALLBACK` | `3600` | Store TTL (seconds) for a token whose `exp` is missing |

Entra ID **access tokens are bearer tokens designed for reuse** within their
lifetime — the ibl.ai platform forwards the same token across many MCP requests —
so blind single-use would break the integration. The modes:
- `bind` (default): on first sight, bind the `jti` to the request's client IP;
  the same `jti` presented from a **different** IP is rejected as replay.
  Legitimate reuse from the platform's stable egress passes; a token stolen and
  replayed from elsewhere is caught. (IP is read from `X-Forwarded-For` set by
  Caddy; deployments with multiple platform egress IPs may need `off` or
  `strict`.)
- `strict`: single-use — any second presentation of a `jti` is rejected. Only for
  deployments that mint one-time tokens.
- `off`: no replay checking.

As with rate limiting, the store is the Django cache: set `ONTOLOGY_CACHE_URL` to
a `redis://…` URL so the check holds across all worker processes (on the default
per-process cache a replay is only caught on the worker that first saw the token).
Replay detection is best-effort; short token lifetimes and sender-constrained
tokens (DPoP / mTLS) remain the strongest defences.

**Vector store authentication:**

| Env var | Default | Purpose |
|---|---|---|
| `CHROMA_TOKEN` | *(required by compose)* | Static bearer token for the ChromaDB vector store |

The `vector-store` container runs ChromaDB's `TokenAuthenticationServerProvider`,
so every request must present `CHROMA_TOKEN`; without it no container on
`ontology-internal` can read or write embeddings. The same value is passed to the
`sync-engine` and `ontology-gateway` services, and the shared `VectorSearch`
client presents it automatically. Generate a strong random value (e.g.
`openssl rand -hex 32`) and set it in the root `.env`; the compose stack fails to
start if it is unset. When `CHROMA_TOKEN` is absent (e.g. a local or air-gapped
run against an unauthenticated store) the client connects without auth.

## Documentation

| Document | Covers |
|---|---|
| [docs/architecture.md](/developer/guides/ontology-architecture) | Full plan: positioning, diagram, seven components, design decisions, rollout |
| [docs/components/01-mcp-inbound.md](https://github.com/iblai/ontology/blob/main/docs/components/01-mcp-inbound.md) | MCP Toolbox + custom MCP servers, `tools.yaml` |
| [docs/components/02-knowledge-materialization.md](https://github.com/iblai/ontology/blob/main/docs/components/02-knowledge-materialization.md) | Sync modes, text memories, Postgres cache, vector index |
| [docs/components/03-identity.md](https://github.com/iblai/ontology/blob/main/docs/components/03-identity.md) · [docs/identity.md](/developer/guides/ontology-identity) | Entra ID flow, `roles.yaml`, Option A vs. B |
| [docs/components/04-mcp-outbound.md](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md) · [docs/platform-integration.md](/developer/guides/ontology-platform-integration) | Outbound MCP server + ibl.ai platform integration |
| [docs/components/05-service-discovery.md](https://github.com/iblai/ontology/blob/main/docs/components/05-service-discovery.md) | Safety suite, introspection, BYOK LLM, adapters |
| [docs/read-only-db-user.md](/developer/guides/ontology-read-only-db-user) | Provisioning a read-only Postgres role for a DB source |
| [docs/components/06-provisioning.md](https://github.com/iblai/ontology/blob/main/docs/components/06-provisioning.md) | The 6-step idempotent pipeline |
| [docs/components/07-cli.md](https://github.com/iblai/ontology/blob/main/docs/components/07-cli.md) | Full CLI reference |
| [docs/deployment.md](/developer/guides/ontology-deployment) | Docker Compose stack, networks, Caddyfile, `.env`, rollout |

## License

MIT — see [LICENSE](https://github.com/iblai/ontology/blob/main/LICENSE).
