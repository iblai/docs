# Where every piece of data comes from

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/data-sources.md`](https://github.com/iblai/ontology/blob/main/docs/data-sources.md). This page is generated — edit it in the repository, not here.

Scope: everything dynamic the console renders, and where each thing it writes is saved.

## The rule

This Next.js app has **no database**. Like the OS app, it keeps nothing of its own beyond the files
the server runtime reads. Every value on screen comes from one of three places, and nothing else:

| Tag     | System                   | Where it lives                                                                     | How the browser reaches it                                                                                                 |
| ------- | ------------------------ | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **IBL** | ibl.ai platform API      | ibl.ai                                                                             | Directly from the browser with the SSO session (`axd_token` / `dm_token` from login.ibl.ai), exactly as os.ibl.ai does.    |
| **AO**  | NVIDIA auto-ontology     | One per platform, own Postgres + pgvector; host's own `/api/*` on `127.0.0.1:3100` | Through this app's route handlers, which add the engine's service key (`x-api-key`). Ports 3001 and 3002 are never called. |
| **TB**  | the host and MCP Toolbox | Files under `$ONTOLOGY_HOME/platforms/<platform>/`; one Toolbox process per server | Through this app's route handlers.                                                                                         |

The host is the platform's own server, the same process that serves the pages
([install.md](/developer/guides/ontology-install)). A console hosted elsewhere (the pages-only build) serves pages only and
forwards every route handler call to the platform's server ([network.md](/developer/guides/ontology-network)).

If something was not available from one of these, it was removed from the UI or shows “—”. The
removals are listed at the end.

The engine gives no row counts, no primary keys, no ingest status and no log over HTTP. Rows show
“—”. A key column is the one other tables' foreign keys point at. A system is “ingesting” until its
tables appear.

## Where each thing is saved

| Thing                                                            | Saved in                                                                                          | Notes                                                                                                                                                            |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Connected databases                                              | the engine's own Postgres (the `postgres` service of its compose stack, database `auto_ontology`) | The whole connection, password included, as unencrypted JSON on the catalog row. Set the engine's four `VAULT_*` variables to keep credentials in Vault instead. |
| The same database password, for serving                          | `env` on the host, mode 0600                                                                      | Toolbox needs it to run tools, and the engine never gives a password back.                                                                                       |
| Custom HTTP APIs                                                 | `apis.yaml`; the key in `env`                                                                     | A file, as Toolbox reads it. The engine does not take APIs.                                                                                                      |
| Terms, tables, joins, tags, certification                        | the engine's Postgres + pgvector                                                                  | Its semantic layer.                                                                                                                                              |
| MCP servers (config, flags, revisions)                           | `servers/<slug>/`                                                                                 | Files. The list of running processes is the pid files.                                                                                                           |
| Tool calls and refusals the gateway saw                          | `servers/<slug>/gateway.log`                                                                      | A log beside Toolbox's own `toolbox.log`. The Runtime Log, the call counters and the Activity table read it.                                                     |
| Roles and policies, API keys, the server registered for an agent | ibl.ai                                                                                            | Written from the browser with the admin's own session.                                                                                                           |

The files are in the platform's own folder, `$ONTOLOGY_HOME/platforms/<platform>/`, and the
engine is the platform's own ([network.md](/developer/guides/ontology-network)). `ONTOLOGY_HOME` defaults to `.ontology`
in the console's directory.

## Screen by screen

### Shell

| Element                                                             | Source        | Detail                                                                                                                                                                                        |
| ------------------------------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Logo, header, sidebar                                               | static        | ibl.ai brand assets.                                                                                                                                                                          |
| Platform menu                                                       | IBL           | The platforms the user belongs to.                                                                                                                                                            |
| User / Admin switch, avatar                                         | IBL           | Session and RBAC (admin mode as in the OS).                                                                                                                                                   |
| Notifications, Invites, Management, API, Secrets, Advanced, Support | IBL           | The OS's own panels.                                                                                                                                                                          |
| Sidebar counts                                                      | AO / TB / IBL | Systems = engine connections; catalog = tables the engine read; ontology = engine terms; MCP gateway = tools across the servers' `tools.yaml`; agents = ibl.ai agents with a server attached. |

### Dashboard

| Element                | Source                                                                                              |
| ---------------------- | --------------------------------------------------------------------------------------------------- |
| Systems connected      | AO `GET /api/connections`                                                                           |
| Object types certified | AO `GET /api/terms` (`name_certified`, `description_certified`)                                     |
| Tools published        | TB tools across the servers' `tools.yaml`                                                           |
| Agent tool calls · 24h | TB each server's `gateway.log`                                                                      |
| Activity               | TB deploys (server revisions); AO compiles and certifications; IBL agent connections                |
| MCP server cards       | TB `servers/<slug>/` (revision, toolsets, tools, process status); IBL agents attached to the server |

### Systems (step 1)

| Element              | Source                                                                                                                                                                                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Connected databases  | AO `GET /api/connections`. The name is the database's name; the URL never shows the password.                                                                                                                                                           |
| Table and PII counts | AO `GET /api/exploration/graph` and the column tags the PII detector sets                                                                                                                                                                               |
| Ingest status        | Derived: “ingesting” until the system's tables appear, “compiling” while AO `GET /api/semantic-compilation/status` says a compile runs                                                                                                                  |
| HTTP API systems     | TB `apis.yaml` (served only, never dissected)                                                                                                                                                                                                           |
| Filesystem systems   | TB `locals.yaml`: a folder or file under `ONTOLOGY_LOCAL_ROOT`, served by the gateway itself as the `<name>-files` toolset; never dissected                                                                                                             |
| Catalog of kinds     | static: what the engine dissects and what Toolbox serves                                                                                                                                                                                                |
| Test Connection      | AO `POST /api/connections/test`                                                                                                                                                                                                                         |
| Connect & Dissect    | The password to TB `env`; AO `PUT /api/configurations/pii-detection` and `…/distinct-value-probing`; AO `POST /api/connections`, which starts the ingest                                                                                                |
| Fetch Spec           | The host fetches the OpenAPI document under the same address rule as the API's base address (the host's own machine outside production only), with the auth header when the document is on the API's own address, and lists its operations as endpoints |
| Add & Expose         | TB `apis.yaml`, the key in `env`                                                                                                                                                                                                                        |
| Prebuilt             | One raw toolset per connected database (`<system>-raw`); the ticked ones open the server builder already ticked                                                                                                                                         |
| Database credentials | Entered once. Stored by the engine (its connection row, or Vault) and in TB `env` for serving. Never in the browser after submit, never on ibl.ai.                                                                                                      |

### Map (step 2)

| Element                                 | Source                                                                                                                                                                  |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pipeline stages                         | Derived from what the engine has read: tables, columns, terms, links, compile finished (AO `GET /api/semantic-compilation/status`), tools published (TB server configs) |
| Log                                     | The changes the console observed in those readings while the page was open. The engine streams no log.                                                                  |
| Tables, foreign keys, system clusters   | AO `GET /api/exploration/graph`                                                                                                                                         |
| Terms, term ↔ table mapping, term links | AO `GET /api/exploration/semantic-graph` (a link is two terms sharing a table)                                                                                          |
| Table detail                            | AO `GET /api/columns/{table_id}`, `GET /api/exploration/tables/{table_id}/details`. Rows show “—”.                                                                      |
| Term detail                             | AO `GET /api/terms/{term_id}`, `GET /api/exploration/terms/{term_id}/details`, `GET /api/terms/{term_id}/sql-attributes`                                                |
| "Same student, three systems" banner    | A term whose tables sit in several databases (a fact, not a match score)                                                                                                |
| Certify                                 | AO `PATCH /api/terms/{term_id}` (`name_certified`, `description_certified`)                                                                                             |
| Re-run                                  | AO `PUT /api/configurations/semantic-compilation` `{enabled: true}`, which starts a compile                                                                             |

### Ontology (step 3)

| Element                             | Source                                                                                                                       |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Object types                        | AO `GET /api/terms`                                                                                                          |
| Edit                                | AO `PATCH /api/terms/{term_id}` (name, description)                                                                          |
| Synonyms                            | AO term `synonyms`, read only: the engine does not take synonyms over its API                                                |
| Properties (backing column, status) | AO `GET /api/terms/{term_id}/column-attributes`; certify with `PATCH …/column-attributes/{attr_id}`                          |
| PII flag                            | AO column tags                                                                                                               |
| Link types                          | AO semantic graph links                                                                                                      |
| Metrics                             | AO `GET /api/terms/{term_id}/sql-attributes` (`expression`, `certified`); certify with `PATCH /api/sql-attributes/{attr_id}` |
| Tags                                | AO `GET /api/tags`                                                                                                           |
| Actions (write tools)               | TB the platform's `actions.yaml`, written by hand; its toolsets count as writing                                             |

### MCP Gateway · new server (step 4)

| Element                      | Source                                                                                                                                                                                                                                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Server name, endpoint        | TB `servers/<slug>/`; the endpoint is `<console origin>/<platform>/<slug>/mcp` (`NEXT_PUBLIC_APP_URL` when set), `https://<host>/<platform>/<slug>/mcp` on the platform's server; a console hosted elsewhere shows the server's second address, `https://<host>/iblai/<platform>/<slug>/mcp` |
| Toolsets and tools           | Generated by the host from certified AO terms and SQL attributes, the connected systems, the custom APIs and `actions.yaml`                                                                                                                                                                  |
| Issue Token                  | IBL: a role and a policy per chosen toolset, made if missing, then an API key limited to those policies. The secret is shown once.                                                                                                                                                           |
| Entra / Google / custom OIDC | TB `authServices` in `tools.yaml`, verified by Toolbox                                                                                                                                                                                                                                       |
| Allowed origins / hosts      | TB `flags.json`, passed to Toolbox as `--allowed-origins` and `--allowed-hosts`; the gateway checks origins too                                                                                                                                                                              |
| Generated config preview     | TB the file the host would write                                                                                                                                                                                                                                                             |
| Save Draft / Deploy          | TB `tools.yaml`, `flags.json` and a new revision. Deploy also starts Toolbox and waits until it has loaded that file. Nothing is registered on ibl.ai.                                                                                                                                       |

### MCP Gateway · server (step 5)

| Element                                  | Source                                                                                                                                                               |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Enabled, revision, Roll Back             | TB `flags.json` and `revisions/`                                                                                                                                     |
| Agent tool calls · 24h, most called tool | TB `gateway.log`                                                                                                                                                     |
| Agents attached                          | IBL agents that have an MCP server registered under this server's endpoint                                                                                           |
| Attach                                   | IBL: a role and policy per chosen toolset, an API key limited to them, an MCP server registered for that agent with the key, and the agent's settings told to use it |
| Playground                               | The gateway at the server's endpoint, with the signed-in user's session                                                                                              |
| Activity: agent tool calls               | TB `gateway.log` (time, tool, agent, result, duration). User and session show “—”.                                                                                   |
| Activity: runtime log                    | TB `gateway.log`                                                                                                                                                     |
| Tokens                                   | IBL API keys that carry an `Ontology:` policy (name, policies, groups, created, expires); Revoke deletes the key                                                     |

### Agents, Roles, Access, Catalog, Compilation, Config

| Screen                   | Source                                                                                                                                                                            |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agents                   | IBL agents and their MCP servers (`auth_type`, scope); call counts from TB `gateway.log`                                                                                          |
| Roles                    | IBL RBAC policies named `Ontology: <toolset>`                                                                                                                                     |
| Access                   | IBL holders of each policy, and the agents connected                                                                                                                              |
| Catalog                  | AO `GET /api/datasources/dbs`, `GET /api/schemas/{db_id}`, `GET /api/tables/{schema_id}`, `GET /api/columns/{table_id}`                                                           |
| Compilation (was "Sync") | AO `GET /api/semantic-compilation/status`; `GET` and `PUT /api/configurations/{semantic-compilation,pii-detection,distinct-value-probing}`. Run now is the compile switch set on. |
| Config                   | TB files and revisions per server; AO `POST /api/model/export`, which needs the service key of an engine admin                                                                    |

## Reaching the host from the browser

The browser talks to two places:

1. **ibl.ai**, directly, with the SSO session. Same as the OS app.
2. **The ontology host**, through this app's own route handlers (`/api/host/*`). They run on the
   platform's server, next to the engine and the Toolbox processes, behind the same TLS proxy. On
   a console hosted elsewhere (the pages-only build) they serve no data: each call names the
   platform's server in `x-ontology-server`, and the console forwards it to the server's second
   address, `https://<host>/iblai/api/host/*`, with the same sign-in, only to a public https
   address, following no redirect. The browser keeps that address per platform in
   `localStorage["ontology_server:<platform>"]` ([network.md](/developer/guides/ontology-network)).

The route handlers keep no state of their own:

- Every request carries the ibl.ai session (`Authorization: Token <dm_token>`) and its platform
  (`x-ontology-platform: <platform key>`). The handler asks ibl.ai what that session is granted on
  that platform and lets in a platform admin or a holder of the console grant. ibl.ai refuses a
  session that does not belong to the platform. A server with an engine of its own refuses every
  platform but the one its `ONTOLOGY_PLATFORM` names before it asks ibl.ai. Details in
  [auth.md](/developer/guides/ontology-auth).
- `/api/host/ao/*` forwards an allow-list of methods and paths to the platform's engine with its
  service key. The engine keeps the data.
- `/api/host/systems`, `/api/host/graph` and `/api/host/toolsets` read the engine once per page load
  and keep that reading a few seconds.
- `/api/host/servers/*` reads and writes the platform's `servers/<slug>/`, starts and stops the
  Toolbox process for that slug, rolls back a revision, and reads the gateway's log.

Agent traffic uses a second path, straight to the platform's server:

- `https://<host>/<platform>/<slug>/mcp` (and the second address,
  `https://<host>/iblai/<platform>/<slug>/mcp`) is the **gateway**. It refuses a credential sent
  over plain HTTP (except on loopback), checks `Origin` and `Host` against the server's lists, asks
  ibl.ai what the caller's credential is granted on that platform (ibl.ai refuses a credential of another platform), limits the call
  to the toolsets that grant reaches, and forwards to Toolbox on `127.0.0.1:<port>`. A Filesystem
  source's two tools it answers itself, not through Toolbox. It writes
  only its own log. On a console hosted elsewhere, its own `/<platform>/<slug>/mcp` answers `404`.
- Toolbox listens on loopback only, so every call goes through the gateway. An OIDC ID token rides
  along in the header Toolbox expects (for example `entra_token`) and Toolbox verifies it.
- When an ibl.ai agent calls the server, ibl.ai sends the API key stored in that agent's MCP server
  on ibl.ai; the gateway sees an ordinary API key.

## Removed or replaced

| Was                                                             | Problem                                                        | Now                                                                       |
| --------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Multiple "ontologies" (Institution, LMS Analytics, Advancement) | no system of record; an engine is one semantic layer           | one ontology per ibl.ai platform, each on its own engine                  |
| DuckDB snapshot of HTTP APIs so they could be dissected         | neither the engine nor Toolbox does this                       | APIs are served only (Toolbox `http`)                                     |
| "7 write operations must fail" read-only proof                  | not a feature of the engine, Toolbox or ibl.ai                 | the engine's connection test; a hint to use a read-only login             |
| 97.8% cross-system match score, "accept merge"                  | fabricated; the engine maps terms to tables, it does not score | the fact of a term spanning N databases; certify the term                 |
| Interfaces (Person, Billable)                                   | not in the engine                                              | engine tags                                                               |
| Row counts, ingest progress, the engine's service log           | not available over the engine's API                            | “—” for rows; stages and log lines derived from what the engine has read  |
| Synonym “+ add”                                                 | the engine's API does not write synonyms                       | synonyms are shown, not added                                             |
| Agent analytics from ibl.ai (user, session per call)            | no verified endpoint                                           | the gateway's own log; user and session show “—”                          |
| p50 latency, authorized %                                       | no store                                                       | removed                                                                   |
| Key prefix, last used, calls per key                            | not in ibl.ai API keys                                         | name, policies, groups, created, expires                                  |
| Rate limit per key                                              | not settable in Toolbox or ibl.ai                              | the gateway limits each caller to 120 requests and 30 tool calls a minute |
| "Claude Desktop · OIDC" row under Agents                        | not an ibl.ai agent                                            | removed; external clients show only in the Runtime Log                    |
| roles.yaml, sync-schedules.yaml                                 | artifacts of an older backend                                  | ibl.ai RBAC policies; the engine's compilation settings                   |
| Masked emails in playground rows                                | masking was attributed to a role policy that does not exist    | rows as returned by the tool                                              |
