# Deployment

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/deployment.md`](https://github.com/iblai/ontology/blob/main/docs/deployment.md). This page is generated — edit it in the repository, not here.

> Part of the iblai-ontology architecture. See the architecture overview.

iblai-ontology deploys as a **Docker Compose stack** on university infrastructure. The stack contains *only* the knowledge layer — the data stores, the MCP servers, the identity gateway, and the MCP exposure endpoint. **The agent runtime is not part of the stack**; it connects remotely over MCP (see [platform-integration.md](/developer/guides/ontology-platform-integration)).

There are two stacks. A server that works with the ibl.ai platform and the hosted console is installed with one command, from a published image ([Install on Ubuntu](#install-on-ubuntu)). The repository's `docker-compose.yml` is the developer stack, with the MCP Toolbox and the connector containers ([The Stack](#the-stack) and everything after it).

---

## Install on Ubuntu

For a platform's administrator who has a fresh server and no wish to clone anything. The same install step by step, with what the script prints and every setting explained, is [install.md](https://github.com/iblai/ontology/blob/main/docs/install.md). The hosted console's **Connect a backend** screen shows this command with the host name filled in:

```bash
curl -fsSL https://github.com/iblai/ontology/releases/latest/download/install.sh \
  | sudo bash -s -- \
    --host ontology.example.edu
```

**Before.** Ubuntu 22.04 or later on amd64. A host name whose DNS record points at the server. Ports 80 and 443 open to the internet: 80 for the certificate and the redirect to https, 443 for everything else. The hosted console and the ibl.ai platform call from addresses that change, so they cannot be allow-listed; every call is guarded by sign-in instead. On the platform: ontology records (`ibl_ai_mentor.0415_ontologyrecord`) and RBAC, which keeps the roles.

**What it does**, in order:

1. Checks it runs as root on Ubuntu 22.04 or later, amd64, and stops with the reason otherwise.
2. Uses the Docker that is there when `docker compose version` works. With none, installs `docker.io` and `docker-compose-v2` from Ubuntu's own archive: no repository and no key file is added. A Docker without Compose stops the script, because mixing Ubuntu's packages with Docker's own breaks both.
3. Asks for the host name if the command did not give one. It takes no key and no platform: the platform is chosen in the frontend, when the link is opened.
4. Asks the platform two things with no key, before anything is written; a token that is none makes the platform itself refuse, in its own words. First `GET /api/core/token/verify/`: a 401 or 403 that carries the platform's `detail` shows this server reaches it, and one without comes from the edge in front of it and stops the script. Second `GET /v1/ontology/kinds/backend/records`: a route that is there answers 401 or 403, and the connected systems and the schedules live in the platform (`ONTOLOGY_DATA_STORE=dm`); a 404 means the platform keeps no ontology records, so they live on this server and only the roles are the platform's (`rbac`). The script says which.
5. Writes `/opt/iblai-ontology/.env`, readable by root alone: the host name, the platform's address, where the settings live, the code of the link (`ONTOLOGY_SETUP_CODE`), and four secrets it generates (the database password, the search index's token, Django's secret and the key that encrypts the logins of the connected systems and the backend's own key).
6. Downloads `compose.yaml` of the same release beside it, pulls the images and starts them.
7. Waits up to three minutes for `https://<host>/mcp` to answer `405`, which the gateway does before any sign-in. If it does not, the last lines of the proxy and the gateway are printed.
8. Prints the link that sets the server up, `https://ontology.ibl.ai/#host=<host>&code=<code>`, the address agents call (`https://<host>/mcp`) and where its two files are. `--frontend <address>` names another frontend for the link; the command a frontend shows carries it when that frontend is not at `https://ontology.ibl.ai`.

**Setting it up.** A server with a code and no platform is locked: the console's API answers 503, `/mcp` answers 503 and the worker's tick queues nothing. Two routes answer. `GET /setup` tells anyone `{"set_up": false}`, which is how the script knows to print the link. `POST /setup` takes `{platform, code}` with an administrator's own sign-in (`Authorization: Token …`):

1. The code must equal `ONTOLOGY_SETUP_CODE`.
2. The platform is asked, uncached, who the caller is and whether they are an active administrator of the platform named (`/api/core/token/verify/`, `/api/core/users/platforms/`).
3. `GET /v1/rbac/permissions/?resource_paths=/custom/ontology/` with the caller's sign-in and `iblai-org: <platform>`: the platform refuses a sign-in of another platform, and the answer must hold `*` or `Ontology/*`, or the platform's administrators cannot grant ontology roles yet.
4. `POST /api/core/platform/api-tokens/` with the caller's sign-in makes the backend's key, in owner mode: it acts as the administrator who set the server up.
5. The platform and the key, encrypted with `ONTOLOGY_CREDENTIAL_KEY`, are kept as one row of the server's database (`services.Record`, kind `server`). The row is created once, so a second caller is refused; every process reads it, so nothing restarts.

In the hosted console the link's host name and code sit after `#`, so they reach neither the console's server nor its logs. The page keeps them in the tab's session storage, which sign-in and a platform switch leave alone, and shows the **Set up** card: the platforms the user administers, none preselected when there are several, and the SDK's own switch for one they are not signed in to, since a key is made in the platform of the sign-in that asks. **Set up** is `PUT /api/hosted/backend` with `{url, platform, code}`; the console's server posts `/setup` to the backend and then asks `/counts` as on every Connect. A backend older than this answers 404 to `/setup` and is connected as before.

A server that is set up takes `POST /setup` from its own platform's administrators alone and ignores the code: it does nothing while the platform takes the key it holds, and makes a new one when the platform refuses it. So **Connect** on the console's Backend page is also how a dead key is replaced.

A server installed with a key keeps the platform and the key its `.env` names (`IBLAI_ORG`, `IBLAI_ADMIN_TOKEN`), and none of the above applies to it. To bring one over, take those two lines out of `.env`, add `ONTOLOGY_SETUP_CODE=<32 hex characters>`, apply it with `docker compose … up --detach` and open `https://<frontend>/#host=<host>&code=<code>`; its old key stays in the platform until deleted there.

**What runs** (`deploy/compose.yaml`):

| Service | What | Reaches |
|---|---|---|
| `proxy` | Caddy: gets and renews the certificate, sends http on to https | The only one that listens on the server |
| `ontology-gateway` | The MCP gateway and the console's API | The platform, the connected systems |
| `sync-engine` | The Celery worker with its scheduler | The platform, the connected systems |
| `migrate` | Database migrations, once at every start | The cache database |
| `ontology-db` | PostgreSQL 16: the synced copy, sync runs, the audit log | Nothing outside |
| `redis` | The worker's queue, kept in memory only | Nothing outside |
| `vector-store` | ChromaDB: the search index | Nothing outside |

Not there: the MCP Toolbox and the connector containers of the developer stack. The gateway's own tools and Twenty CRM's operations do not go through them, so on such a server an agent can query the synced copy and change Twenty. A database can be connected and synced as well, since the sync reads it through its own connection, and agents query its synced copy; a tool that reads or changes the database itself goes through the Toolbox and answers an error here.

**What it leaves on the server.** The Docker packages, if it installed them. `/opt/iblai-ontology` with `compose.yaml` and `.env`. Docker's volumes (the database, the memory files, the search index, the config directory, the certificates) and images. No language runtime, no source tree, nothing in a home directory, no firewall change.

**Keep a copy of `.env`.** The logins of the connected systems are kept encrypted, in the platform or in this server's database, and so is the backend's own key; `ONTOLOGY_CREDENTIAL_KEY` in that file is what decrypts them. Without it they must be entered again. The `db` volume holds the platform the server was set up for and that key, and where the server keeps its own settings (`rbac`) the connected systems and the list of connected agents; the `config` volume holds the schedules. They are this server's to back up.

**Day to day.**

```bash
docker compose -f /opt/iblai-ontology/compose.yaml ps
docker compose -f /opt/iblai-ontology/compose.yaml logs --follow ontology-gateway
```

- **Upgrade:** run the install command again. An existing `.env` is kept and nothing is asked, so where the settings live does not change; the new release's `compose.yaml` and images replace the old ones, and the migrations run. On a server that is not set up yet it prints the link again.
- **Remove:** the same command with `--uninstall`. It asks for the host name to be typed, then removes the containers, the volumes, the images and the directory. Docker itself stays, and so does what the platform keeps: the roles, the key and MCP server of each agent still connected, the backend's API key, and with `dm` the schedules, the connected systems and the backend's address. With `rbac`, remove each agent's connection in the console first: the list goes with the server.
- **Preview:** `--dry-run` says each step and changes nothing.

Then open the link, choose the platform and press **Set up**; add a system, and connect an agent on the **Agents** page. Connecting Twenty CRM makes the role an agent is connected as; for a database, give a role its tables and toolsets first. From another computer, enter the host name under **Connect a backend** and press Connect.

---

## Prerequisites

For the developer stack below.

The university provides:

- A Linux server or VM inside their network — **Ubuntu 22.04+, 16 GB RAM minimum, 500 GB disk**.
- **Docker + Docker Compose** installed.
- Network access from that server to the source systems: PeopleSoft Oracle (port 1521), and HTTPS to Canvas / Slate / Navigate APIs.
- A **read-only** database user with SELECT grants on the relevant schemas (the [safety suite](https://github.com/iblai/ontology/blob/main/docs/components/05-service-discovery.md#the-read-only-safety-verification-suite) will verify this).
- A **Canvas admin API token** and **Slate API credentials** (as applicable).
- An **Entra ID app registration** for iblai-ontology (see [identity.md](/developer/guides/ontology-identity#entra-id-app-registration-the-minimum) — only basic auth scopes, no App Roles needed).
- A **DNS record** pointing `ontology.<university>.edu` at the server.

---

## The Stack

`docker-compose.yml` (knowledge layer only). Annotated and condensed; the canonical file is generated by `ontology config init` and extended by the [provisioning engine](https://github.com/iblai/ontology/blob/main/docs/components/06-provisioning.md).

```yaml
services:
  # --- MCP Inbound: database connectivity (Component 1) ---
  mcp-toolbox:
    image: us-central1-docker.pkg.dev/database-toolbox/toolbox/toolbox:1.5.0
    volumes: [./config/tools.yaml:/app/tools.yaml]
    env_file: .env.mcp
    networks: [ontology-internal]
    restart: unless-stopped

  # --- MCP Inbound: custom API servers (one per source) ---
  mcp-canvas:   { build: ./mcp-servers/canvas,   env_file: .env.canvas,   networks: [ontology-internal], restart: unless-stopped }
  mcp-slate:    { build: ./mcp-servers/slate,     env_file: .env.slate,    networks: [ontology-internal], restart: unless-stopped }
  mcp-navigate: { build: ./mcp-servers/navigate,  env_file: .env.navigate, networks: [ontology-internal], restart: unless-stopped }
  mcp-ldap:     { build: ./mcp-servers/ldap,      env_file: .env.ldap,     networks: [ontology-internal], restart: unless-stopped }

  # --- Knowledge store (Component 2) ---
  ontology-db:
    image: postgres:16-alpine
    volumes:
      - ontology-data:/var/lib/postgresql/data
      - ./sql/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql
      - ./sql/views.sql:/docker-entrypoint-initdb.d/02-views.sql
    environment:
      POSTGRES_DB: ontology
      POSTGRES_USER: ontology
      POSTGRES_PASSWORD: ${ONTOLOGY_DB_PASSWORD}
    networks: [ontology-internal]
    restart: unless-stopped

  vector-store:
    image: chromadb/chroma:0.6.3
    volumes: [vector-data:/chroma/chroma]
    environment:
      ANONYMIZED_TELEMETRY: "false"
      CHROMA_SERVER_AUTHN_PROVIDER: chromadb.auth.token_authn.TokenAuthenticationServerProvider
      CHROMA_SERVER_AUTHN_CREDENTIALS: ${CHROMA_TOKEN:?set CHROMA_TOKEN in .env}
    networks: [ontology-internal]
    restart: unless-stopped

  # --- The sync worker's queue ---
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--save", "", "--appendonly", "no"]
    networks: [ontology-internal]
    restart: unless-stopped

  # --- Sync engine (Component 2; Django + Celery): the one image, with the worker's command ---
  sync-engine:
    build: .
    command: [celery, -A, "iblai_ontology.backend.celery:app", worker, --beat, --loglevel=info, --schedule=/tmp/celerybeat-schedule]
    volumes:
      - ontology-files:/ontology
      - ./config:/app/config:ro              # the config directory, read-only
    environment:
      MCP_TOOLBOX_URL: http://mcp-toolbox:5000
      MCP_CANVAS_URL: http://mcp-canvas:3000
      ONTOLOGY_DB_URL: postgresql://ontology:${ONTOLOGY_DB_PASSWORD}@ontology-db:5432/ontology
      ONTOLOGY_CELERY_BROKER: redis://redis:6379/0
      CHROMA_URL: http://vector-store:8000
      CHROMA_TOKEN: ${CHROMA_TOKEN:?set CHROMA_TOKEN in .env}
    depends_on: [mcp-toolbox, mcp-canvas, ontology-db, redis, vector-store]
    networks: [ontology-internal, ontology-exposed]   # reaches the platform and remote sources
    restart: unless-stopped

  # --- MCP Outbound: exposure gateway (Component 4): the same image, its default command ---
  ontology-gateway:
    build: .
    volumes:
      - ontology-files:/ontology:ro          # read-only mount
      - ./config:/app/config                 # writable: the Ontology Frontend saves through the gateway
    environment:
      MCP_TOOLBOX_URL: http://mcp-toolbox:5000
      ONTOLOGY_DB_URL: postgresql://ontology:${ONTOLOGY_DB_PASSWORD}@ontology-db:5432/ontology
      ONTOLOGY_CELERY_BROKER: redis://redis:6379/0   # syncs asked for in the console go to the worker
      CHROMA_URL: http://vector-store:8000
      CHROMA_TOKEN: ${CHROMA_TOKEN:?set CHROMA_TOKEN in .env}
      ENTRA_TENANT_ID: ${ENTRA_TENANT_ID}
      ENTRA_CLIENT_ID: ${ENTRA_CLIENT_ID}
    depends_on: [mcp-toolbox, ontology-db, redis, vector-store]
    networks: [ontology-internal, ontology-exposed]   # bridges both networks
    restart: unless-stopped

  # --- HTTPS reverse proxy ---
  proxy:
    image: caddy:2-alpine
    volumes:
      - ./config/Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data
    ports: ["443:443"]
    depends_on: [ontology-gateway]
    networks: [ontology-exposed]
    restart: unless-stopped

networks:
  ontology-internal:
    driver: bridge
    internal: true            # No external/internet access for internal services
  ontology-exposed:
    driver: bridge            # Gateway and proxy can reach the outside

volumes:
  ontology-data:
  ontology-files:
  vector-data:
  caddy-data:
```

### Internal vs. exposed networks

This is the heart of the on-prem security posture:

- **`ontology-internal`** is marked **`internal: true`** — containers on it **cannot reach the internet**. The databases, Redis, the MCP Toolbox and the custom MCP servers live here only.
- **`ontology-exposed`** is a normal bridge that can reach outside.
- **`ontology-gateway`** and **`sync-engine`** are on both: they reach the ibl.ai platform and the sources outside the stack. Only **`proxy`** publishes a port. So PeopleSoft credentials, the cache, and the memory files are never directly exposed; the only door in is the gateway, behind the firewall + Entra-token gating described in [Component 4](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md).

Note also the gateway mounts `ontology-files` **read-only** (`:ro`) — the exposure layer can serve memory files but never modify them. That is why a sync asked for in the console is queued for the worker, through Redis, instead of running in the gateway.

---

## Caddyfile

`config/Caddyfile` — Caddy auto-provisions TLS via Let's Encrypt (or use a university-provided certificate):

```
ontology.alasu.edu {
    reverse_proxy ontology-gateway:8080
    log { output file /var/log/caddy/access.log }
}
```

---

## Environment File

Credentials are **split per MCP server** for isolation — each container loads only its own file via `env_file:`. Compose will not parse unless **every** referenced file exists, and each custom MCP server does `os.environ["X"]` at import, so a missing/empty file crash-loops that one container. Templates for all of them ship as `.env.*.example` (copy each to the real name; the real files are git-ignored):

| File | Vars | Container |
|---|---|---|
| `.env` | `ONTOLOGY_DB_PASSWORD`, `CHROMA_TOKEN`, `ENTRA_TENANT_ID`, `ENTRA_CLIENT_ID` | compose `${...}` + gateway |
| `secrets/ontology_credential_key` | the registry's Fernet key, one line | ontology-gateway, mounted at `/run/secrets/ontology_credential_key` via `secrets:` |
| `.env.mcp` | `PEOPLESOFT_DB_HOST/PORT/NAME`, `PEOPLESOFT_RO_USER/PASSWORD`, `ONTOLOGY_DB_USER`, `ONTOLOGY_DB_PASSWORD` | mcp-toolbox |
| `.env.canvas` | `CANVAS_BASE_URL`, `CANVAS_API_TOKEN` | mcp-canvas |
| `.env.slate` | `SLATE_BASE_URL`, `SLATE_API_KEY` | mcp-slate |
| `.env.navigate` | `NAVIGATE_BASE_URL`, `NAVIGATE_API_KEY` | mcp-navigate |
| `.env.ldap` | `LDAP_URI`, `LDAP_BIND_DN`, `LDAP_BIND_PASSWORD`, `LDAP_BASE_DN` | mcp-ldap |

The registry key is a **Compose secret**, not an env var: generate it once with
`python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())" > secrets/ontology_credential_key && chmod 600 secrets/ontology_credential_key`,
and export `ONTOLOGY_CREDENTIAL_KEY_FILE=./secrets/ontology_credential_key` for host-side `ontology`
commands so the CLI and the gateway share one key. `secrets/` is git-ignored, and Compose refuses
to start the stack while the file is missing.

`CANVAS_BASE_URL` is the instance root only — **no trailing slash, no `/api/v1`** (the server appends it). The Canvas token must be **admin-scoped** to look up other users; the tools accept a Canvas user id, a SIS id, or a `sis_login_id:` ref.

> **Cloud sources note:** `ontology-internal` is `internal: true` (no internet egress). A cloud source (e.g. `*.instructure.com`) is unreachable from a container placed only on that network — reach it from an egress-capable network, or test the MCP server standalone.

Representative `.env` values:

```bash
# PeopleSoft Oracle (read-only credentials provided by the university)
PEOPLESOFT_DB_HOST=psft-db.internal.alasu.edu
PEOPLESOFT_DB_PORT=1521
PEOPLESOFT_DB_NAME=CSPRD
PEOPLESOFT_RO_USER=iblai_readonly
PEOPLESOFT_RO_PASSWORD=<provided-by-university>

# Canvas LMS
CANVAS_BASE_URL=https://alasu.instructure.com
CANVAS_API_TOKEN=<generated-by-canvas-admin>

# Slate CRM
SLATE_BASE_URL=https://apply.alasu.edu
SLATE_API_KEY=<provided-by-slate-admin>

# Local iblai-ontology DB
ONTOLOGY_DB_PASSWORD=<generated-during-setup>

# Microsoft Entra ID
ENTRA_TENANT_ID=<university-tenant-id>
ENTRA_CLIENT_ID=<iblai-ontology-app-client-id>

# Credential encryption (Fernet) for the service registry: a Compose secret file,
# not an env var — see "Environment File" above.
# ONTOLOGY_CREDENTIAL_KEY_FILE=./secrets/ontology_credential_key
```

### The Ontology Frontend

The Ontology Frontend, the admin console, is not a service of this stack. Run one beside it (its
`ONTOLOGY_API_URL` naming the gateway), or use the one hosted for every platform at
`ontology.ibl.ai` (available soon): set `ONTOLOGY_DM_URL`, `IBLAI_ORG` and the platform key on the gateway, and
let the hosted frontend through the firewall as you do the ibl.ai platform. An administrator who
signs in there connects the backend by entering its host name under **Connect a backend** (the
address is taken to be `https://<host name>`; another one goes under Advanced);
`ontology platform address --url https://ontology.<domain>` on the server records the same
address. The address must be `https` and public, and the frontend sends each admin's platform
token and nothing else ([DEV.md](https://github.com/iblai/ontology/blob/main/DEV.md#one-ontology-frontend-for-every-platform)).

The platform keeps that address as an ontology record. For platforms that keep none yet, the
hosted frontend runs with `ONTOLOGY_BACKEND_STORE=browser`: each administrator's browser keeps
the address and connects once, and nothing is written to the platform.

---

## MCP Toolbox config generation

The `mcp-toolbox` container does **not** read `config/tools.yaml` directly — that
file is the ontology's authoring DSL (`kind: source|tool|toolset` documents). The
Toolbox needs its own native format (top-level `sources` / `tools` / `toolsets`
maps). `ontology mcp build` translates the DSL into
`config/generated/toolbox.yaml` (git-ignored), which the container mounts:

```bash
ontology mcp build          # writes config/generated/toolbox.yaml
# ontology deploy up runs this automatically before starting the stack
```

The gateway also writes that file by itself whenever `tools.yaml` changes: a save from the
frontend, a system connected or removed. The Toolbox watches the file and reloads it, so nothing
is restarted.

Translation rules: `type` → `kind`; Oracle's `database` → `serviceName`; `${VAR}`
tokens are left intact so the Toolbox expands them from `.env.mcp` (secrets never
land in the generated file). Tools whose statement is a `${...}` raw-SQL
passthrough (e.g. `query-ontology-cache`) can't be expressed as a Toolbox
parameterized query — they're served **directly by the gateway** and excluded
from the generated file. The gateway calls tools via the Toolbox `/mcp` JSON-RPC
endpoint (the legacy `/api/tool/<name>` REST path is disabled in Toolbox ≥1.5).

> **Eager connection — important.** The Toolbox connects to **every** source at
> startup and **refuses to start if any source is unreachable**. So the active
> `tools.yaml` must contain only sources you have actually configured and can
> reach. For this reason the higher-ed PeopleSoft sample is kept **out** of the
> default `tools.yaml` — it lives in `config/tools.higher-ed.example.yaml` (paired
> with `config/roles.higher-ed.example.yaml`). Merge it in only when you have a
> real PeopleSoft to point at; otherwise the `mcp-toolbox` container crash-loops.
> A source that no tool uses is left out of the generated file, so a database connected for
> syncing alone (its source carries placeholders and no tool) never stops the Toolbox.

### The Toolbox in the one-command install

`deploy/compose.yaml` runs the Toolbox to call the **custom APIs** an administrator connects in
the frontend (see [Install](https://github.com/iblai/ontology/blob/main/docs/install.md#a-system-with-a-web-api)).

- **Its network.** `toolbox`, shared with the gateway alone. It has a way out, to reach the
  APIs; the database, Redis and the search index are not on it. No port is published.
- **Its file.** The `config` volume, read-only, at `/config/generated/toolbox.yaml`. `migrate`
  writes it at every start (`ontology mcp build`), so the Toolbox finds one on a new server and
  on one installed before it. The gateway rewrites it on every change, and the Toolbox reloads
  it: it watches the file, and also looks every two seconds (`--poll-interval 2`).
- **No key.** The Toolbox has no environment and its file holds no secret. A custom API's key is
  kept encrypted with its system, and the gateway adds it to each call.
- **One wrong entry stops it whole.** The Toolbox refuses its entire file when one entry is
  wrong: at start it then exits, and on a reload it keeps the file before. So each written file
  holds an empty toolset named after a hash of its content, and after a save the gateway asks
  the Toolbox for that toolset for up to five seconds. A save the Toolbox does not take is put
  back and refused ("the MCP Toolbox did not take it"); `docker compose logs mcp-toolbox` says
  which entry. This check runs wherever `MCP_TOOLBOX_URL` is set, the developer stack included.
- **Where it may call.** A custom API's address is `https://`, or `http://` inside a private
  network. Only a system whose own address is private may reach private addresses
  (`allowedIpRanges` on its sources); this server's own address and link-local ones are refused
  when the system is connected.

### Reaching source databases (local or remote)

Source connections (`client-postgres`, `client-mysql`, …) are host/credential-driven
via `.env.mcp`, so the same config connects to a **local container** or a
**remote host** — only the network path differs. That path is deployment-specific
and lives in a git-ignored `docker-compose.override.yml` (copy from
`docker-compose.override.example.yml`), not the committed compose:

- **Local source containers:** add `mcp-toolbox` to the source's docker network
  and set `*_HOST` to the container name (e.g. `my-postgres`, `my-mysql`).
- **Remote sources:** `ontology-internal` is `internal: true` (no egress), so add
  `mcp-toolbox` to an egress-capable network and set `*_HOST` to the remote
  hostname/IP.

## Gateway hardening (environment variables)

The gateway middleware is tuned entirely through environment variables on the
`ontology-gateway` container — no code change or rebuild, just set and restart.

**Rate limiting** (fixed-window throttle, keyed on the authenticated subject and
falling back to client IP; a batch counts once for each `tools/call` it carries):

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
| `ONTOLOGY_REQUIRE_HTTPS` | `true` | Reject Bearer tokens received over a plaintext (non-HTTPS) connection, and on `/mcp` a `Token` or `Api-Token` credential too |
| `ONTOLOGY_HSTS_MAX_AGE` | `31536000` | `Strict-Transport-Security` max-age (seconds; `0` disables HSTS) |
| `ONTOLOGY_HSTS_INCLUDE_SUBDOMAINS` | `true` | Add `includeSubDomains` to HSTS |
| `ONTOLOGY_CSP` | `default-src 'none'; frame-ancestors 'none'` | `Content-Security-Policy` value |
| `ONTOLOGY_REFERRER_POLICY` | `no-referrer` | `Referrer-Policy` value |
| `ONTOLOGY_FRAME_OPTIONS` | `DENY` | `X-Frame-Options` value |

HSTS is only emitted over HTTPS. TLS terminates at the Caddy edge, so the gateway
trusts the proxy's `X-Forwarded-Proto` (`SECURE_PROXY_SSL_HEADER`) to decide
whether a connection is secure — keep the gateway reachable only via the proxy,
never directly on `:8080`.

**An ibl.ai credential on `/mcp`** (a DM token as `Token`, a platform API key as
`Api-Token`, either behind `Bearer`; it carries its own role, and `X-Iblai-Role` sent
with one is a `400`; [identity.md](/developer/guides/ontology-identity#a-persons-own-iblai-credential)):

| Env var | Default | Purpose |
|---|---|---|
| `ONTOLOGY_DM_URL` | unset | Where the platform is asked what a credential is granted. Unset, none is taken |
| `IBLAI_ORG` | unset | The one platform a credential is judged on. Unset, none is taken. `ONTOLOGY_ADMIN_PLATFORMS` widens the console only |
| `ONTOLOGY_ROLES_TTL` | `60` | Seconds a credential's grants, or its refusal, are kept: a role given, a membership ended or a key deleted shows within it |
| `ONTOLOGY_DM_VERIFY_TTL` | `300` | Seconds the name and standing of a credential's owner are kept |
| `ONTOLOGY_GATEWAY_URL` | unset | The address agents reach the gateway at. The console's Agents page registers it on an agent, and refuses without it |
| `ONTOLOGY_DATA_STORE` | `local` | Where the settings live. `local`: the files and this database. `dm`: the platform keeps the roles, the schedules and the registry. `rbac`: the platform's RBAC keeps the roles and this host the rest, for a platform without ontology records. Only with `rbac` or `dm` is anyone given a role in the platform, or an agent connected |

**The sync worker:**

| Env var | Default | Purpose |
|---|---|---|
| `ONTOLOGY_CELERY_BROKER` | unset | A `redis://…` URL, the same on the gateway and the worker: syncs asked for in the console, and the refresh after an agent's change, are queued for the worker. Unset, a task runs where it is asked for, which fits a local backend and not a gateway whose memory files are read-only |

**Tools that change data** need nothing here: the write account's variables go in
the Toolbox's environment (`.env.mcp`) beside the read-only ones, and an `http`
source on a private address lists it in `allowedIpRanges`
([components/01-mcp-inbound.md](https://github.com/iblai/ontology/blob/main/docs/components/01-mcp-inbound.md#tools-that-change-data)).
Twenty CRM's operations use the key the system was connected with, kept encrypted in
the registry.

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

## Operating the Stack

The `ontology deploy` commands wrap Docker Compose (see [CLI reference](https://github.com/iblai/ontology/blob/main/docs/components/07-cli.md#ontology-deploy--docker-compose-lifecycle)):

```bash
ontology deploy up --build        # start everything
ontology deploy status            # container status
ontology deploy logs sync-engine -f --tail 20
ontology deploy restart ontology-gateway
ontology deploy down              # stop (add --volumes to wipe data)
```

```
$ ontology deploy status
NAME                STATUS    PORTS
ontology-db         running   5432/tcp
vector-store        running   8000/tcp
mcp-toolbox         running   5000/tcp
mcp-canvas          running   3000/tcp
redis               running   6379/tcp
sync-engine         running
ontology-gateway    running   8080/tcp
proxy               running   0.0.0.0:443->443/tcp
```

Validate the deployment with `ontology health` and `ontology config validate`.

---

## Phased Rollout

### Phase 1 — Foundation (Weeks 1–3)

**Week 1 — Infrastructure and first source.**
1. Clone the deployment repo onto the server; `cp .env.example .env` and fill it in.
2. Start core infra: `docker compose up -d ontology-db vector-store mcp-toolbox`.
3. Verify Toolbox reaches PeopleSoft (call `get-student-enrollment`).
4. Run the initial full sync: `docker compose up -d sync-engine`; watch logs.
5. Verify memory files and the Postgres cache populated (`SELECT COUNT(*) FROM students;`).

**Week 2 — Exposure and authentication.**
6. Configure the Entra ID app registration ([identity.md](/developer/guides/ontology-identity)).
7. Start the gateway + proxy: `docker compose up -d ontology-gateway proxy`.
8. Test token validation with a real Entra token (device-code flow), then verify role scoping: a financial-aid counselor sees finaid + enrollment tools; a student sees only self-service tools; an unassigned user gets the `default` role (or 403 where applicable).

**Week 3 — Connect the agent runtime and validate end-to-end.**
9. Register iblai-ontology as an MCP Server in the ibl.ai platform and attach it to agents ([platform-integration.md](/developer/guides/ontology-platform-integration)).
10. Drive an end-to-end agent interaction: user opens chat → platform triggers in-chat OAuth → user authenticates → platform creates ConnectedService + MCPServerConnection → agent queries the gateway with the user's token → scoped data returned.
11. Open the Ontology Frontend; cross-check ontology data against PeopleSoft reports (counts, GPAs, holds).

### Phase 2 — Expand sources (Weeks 3–6)

12. Deploy the Canvas and Slate MCP servers (`docker compose up -d mcp-canvas mcp-slate`).
13. Extend sync schedules so student memory files include Canvas grades and Slate admissions data.
14. Build and test the Financial Aid Agent and the Advising Agent.
15. Verify role-based boundaries: a counselor sees aid data but not HR; a student sees only their own record.

### Phase 3 — Full iblai-ontology (Weeks 6–10)

16. Add the remaining MCP servers: Navigate, LDAP/AD, and others.
17. Build department agents (IT Help Desk, Facilities, HR).
18. Enable vector search over text memories via ChromaDB.
19. Build out the Ontology Frontend where it is thin (an audit-log viewer).
20. Production-harden: TLS certificates, backup schedules, monitoring/alerting, log aggregation.
21. Load test concurrent users across roles; verify response times and source-system impact.

### Phase 4 — Advanced (ongoing)

22. **Event-driven sync** via webhooks from PeopleSoft / Canvas / Slate for real-time updates to critical data (new holds, registration changes, grade submissions).
23. Student-facing self-service agents with limited scope.
24. Cross-institutional templates — package the stack so a new deployment is *clone → fill credentials → `docker compose up`*.
25. **An approval step for writes** — named write tools ship today (a `write: true` toolset, granted by name); the open item is the queue in front of them: an agent creates a pending action, a human approves in the UI, and the write tool runs.

---

## Related

- The components running in each container: [architecture.md](/developer/guides/ontology-architecture)
- Network/identity gating at the gateway: [components/04-mcp-outbound.md](https://github.com/iblai/ontology/blob/main/docs/components/04-mcp-outbound.md), [identity.md](/developer/guides/ontology-identity)
- Registering with the platform: [platform-integration.md](/developer/guides/ontology-platform-integration)
- Operating the stack from the CLI: [components/07-cli.md](https://github.com/iblai/ontology/blob/main/docs/components/07-cli.md)
