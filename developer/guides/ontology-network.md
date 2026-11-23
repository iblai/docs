# Network and deployment

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/network.md`](https://github.com/iblai/ontology/blob/main/docs/network.md). This page is generated — edit it in the repository, not here.

The console has two halves: its pages, and its server half (the route handlers under
`/api/host/*`, the MCP gateway, the Toolbox processes and the files under `$ONTOLOGY_HOME`). One
codebase runs them four ways:

| Shape                      | Setting                                          | Behaviour                                                                                                                                                |
| -------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The client's server        | `ONTOLOGY_PLATFORM`, with the engine in `AO_URL` | the whole console, built from source on the server, beside NVIDIA's engine and Caddy in Docker ([install.md](/developer/guides/ontology-install)); serves that one platform      |
| A console hosted elsewhere | `NEXT_PUBLIC_ONTOLOGY_RELAY=1` at build          | serves pages only; every `/api/host/*` call is forwarded to the platform's server the browser names in the `x-ontology-server` header                    |
| One host run by hand       | none, or `AO_URL` with `ONTOLOGY_PLATFORM`       | pages and server half as one process: a developer's machine (`pnpm dev`), or one host inside the institution's network behind a TLS proxy                |
| Your own machine           | `ONTOLOGY_PLATFORM`, `ONTOLOGY_HOST` `localhost` | a checkout run without Caddy, in Docker (`--scale proxy=0`) or with none, at `http://localhost:3000` on that machine only ([dev_local.md](https://github.com/iblai/ontology/blob/main/docs/dev_local.md)) |

By default a platform's server serves the whole console at `https://<host>/`. The console is built
there from source and run by systemd, and NVIDIA's engine and Caddy run in Docker
([install.md](/developer/guides/ontology-install)). It needs outbound access to the ibl.ai backend, the data
sources it dissects and serves, and `OPENAI_API_BASE_URL` when a model key is set. Nothing else
but the certificate authority and the host's own hygiene ([below](#outbound-from-the-host)).

## Topology

```
  admin's browser ──443──▶ ┌─────────────────────────────────────────────────────────────────────┐
        │                  │  The platform's server  (e.g. ontology.university.edu)              │
        │ 443              │                                                                     │
        ▼                  │  Caddy :443 (TLS, certificate obtained automatically), :80          │
  ibl.ai (sign-in,         │    /api/host/*                  → console 127.0.0.1:3000            │
  platform data)           │    /api/auth/*                  → 404                               │
                           │    /internal/*                  → 404                               │
  Claude / Codex /         │    /api/*                       → NVIDIA's web app 127.0.0.1:3100   │
  Cursor ──────443────────▶│                                   (its API, with its own keys)      │
                           │    /iblai/api/host/*,           → console, prefix stripped          │
  ibl.ai agents ─443──────▶│    /iblai/<platform>/<slug>/mcp                                     │
  (from ibl.ai egress)     │    everything else              → console: pages, and the gateway   │
                           │                                   at /<platform>/<slug>/mcp         │
                           │                                   → Toolbox 127.0.0.1:<port>        │
                           │                                                                     │
                           │  console → engine 127.0.0.1:3100, with the server's engine key      │
                           │  engine  → console /internal/models/v1, with its model key          │
                           │  loopback only: console 3000, engine web 3100, engine API 3001,     │
                           │                 ingestion 3002, Postgres 5434, every Toolbox port   │
                           │  volumes: secrets, postgres_data, certificates                      │
                           │  the console's files: ONTOLOGY_HOME=/var/lib/ontology,              │
                           │           platforms/<platform>/{env, apis.yaml, servers/<slug>/…}   │
                           └──────────┬────────────────────┬────────────────────┬────────────────┘
                                      │ 443                │ db ports           │ 443, only with
                                      ▼                    ▼                    ▼ OPENAI_API_KEY
                            ibl.ai backend          data sources          OPENAI_API_BASE_URL
                            ext.ibl.ai              SIS Postgres 5432,    (default
                            (129.80.99.227)         LMS MySQL 3306,       api.openai.com)
                                                    Snowflake 443, CRM
                                                    Postgres 5432,
                                                    HTTP APIs 443
```

Agents call `https://<host>/<platform>/<slug>/mcp`. `https://<host>/iblai/…` is a second address
for the console's host API and gateway, used by a console hosted elsewhere; Caddy strips the prefix.

### One platform or any

The engine has no notion of platforms, so each platform has its own, and `ONTOLOGY_PLATFORM` says
which one a server serves.

- A server with an engine of its own (`AO_URL`) serves one platform: the one `ONTOLOGY_PLATFORM`
  names, by its key. Every `/api/host/*` call and every gateway call for another platform answers
  `403` "This server serves another platform." before the server asks ibl.ai anything. The Compose
  install always brings an engine, so it always needs `ONTOLOGY_PLATFORM`.
- `AO_URL` without `ONTOLOGY_PLATFORM`: the server refuses to start, and every host call and
  gateway call, `GET /api/host/health` included, answers `503` "ONTOLOGY_PLATFORM is not set: it
  names the platform this server’s engine (AO_URL) belongs to."
- Neither set: any platform is served, each through its own
  `$ONTOLOGY_HOME/platforms/<folder>/engine.json` (`{"url": "...", "key": "..."}`), written by the
  server's operator. That file names an engine on another machine: one on the same machine is
  refused with a `503` naming the file, because a local engine has one way in, `AO_URL` with
  `ONTOLOGY_PLATFORM`. A platform with no engine gets `503` "No engine is connected for
  “<platform>”. This server’s operator sets AO_URL, or writes platforms/<folder>/engine.json."

### The engine

The server half calls only the engine's web app on `127.0.0.1:3100` (`AO_URL`). It never calls
ports 3001 or 3002. Its key is `AO_API_KEY` when one is set. Without one, the first call that needs
the engine signs in to it as the engine's admin (`AO_ADMIN_EMAIL`, the password in the file
`AO_ADMIN_PASSWORD_FILE`, with the origin the engine trusts, `AO_ORIGIN`, default `AO_URL`'s own
origin), makes a key, keeps it in `$ONTOLOGY_HOME/engine-key` (mode 0600) and uses it from then on.
A key the engine refuses with `401` is dropped, so the next call makes a new one. Nobody has to mint
a key.

Compose runs every service on the host's network (a Linux host), each listening on `127.0.0.1`
only, with the engine's web app on 3100 (3000 is the console's). The engine then reaches a database
on this machine at `127.0.0.1`, the address the runtime uses for it too. The engine's API trusts
its caller's identity header, which is why 3001 and 3002 stay on loopback. The services are
NVIDIA's own images, pulled from `nvcr.io` at the release `compose.yaml` names.

NVIDIA's API is at `https://<host>/api/…`: everything under `/api` but `/api/host` (the console's)
and `/api/auth`, which answers `404` so NVIDIA's sign-in stays off the internet. It takes its own
keys, and its answers carry a sandboxing `Content-Security-Policy`. NVIDIA's own pages are not
served: on one host name they would share an origin with the console and could read its sign-in.
They open through an SSH tunnel, at `http://localhost:3100`, as `admin@ontology.local`:

```bash
ssh -L 3100:127.0.0.1:3100 <server>
sudo docker compose exec postgres cat /secrets/admin-password    # on the server, in /opt/iblai-ontology
```

### Models

The engine reaches a model only through the console. It calls a loopback route of the console,
`POST /internal/models/v1/chat/completions` and `/embeddings`, with a generated key of its own,
checked against the file `ONTOLOGY_MODEL_KEY_FILE` names (unset, the route answers `404`). Caddy
answers `404` for `/internal/*`. With `OPENAI_API_KEY` set, the console swaps in the model name
(`OPENAI_MODEL`, default `gpt-4.1`; `OPENAI_EMBEDDING_MODEL`, default `text-embedding-3-large`),
adds the key, asks for 2048-dimension embeddings and sends the call on to `OPENAI_API_BASE_URL`
(default `https://api.openai.com/v1`). The engine never holds the key.

Without `OPENAI_API_KEY` the route answers `403` and nothing leaves the server: the engine names
terms from names, infers no joins and answers no questions. Connecting a database, the map,
certifying, raw SQL and certified-term tools work either way.

The admin's browser talks to the console and to ibl.ai (`ext.ibl.ai`) directly for sign-in and
platform data, exactly as os.ibl.ai does. So clients need outbound 443 to `ext.ibl.ai` as well.

### A console hosted elsewhere

A console built with `NEXT_PUBLIC_ONTOLOGY_RELAY=1` serves pages only, for platforms whose server
runs elsewhere. ibl.ai hosts one at ontology.iblai.me (staging; ontology.ibl.ai later). It keeps
nothing and holds no secret. Its pages call its own `/api/host/*` with the admin's ibl.ai sign-in,
the platform as `x-ontology-platform` and the platform's server as `x-ontology-server`
(`https://<host>/iblai`). It forwards each call to `https://<host>/iblai/api/host/*` with the same
sign-in, only to a public https address, following no redirect, and gives up after 60 seconds.
Without the header it answers `404` "No ontology server is connected for this platform." Its own
`/<platform>/<slug>/mcp` answers `404`: agents call the server.

The browser keeps each platform's server address itself, in
`localStorage["ontology_server:<platform>"]`; the platform has no record of it. An administrator
connects it by host name under **Connect a server**, which asks the server's
`GET /api/host/health`, or with the server control on **Config**.

### One host run by hand

Pages and server half run as one Next.js process on `127.0.0.1:3000`: `pnpm dev` on a developer's
machine, or behind a TLS proxy (Caddy or nginx) on one host inside the institution's network, with
`/` and `/api/host/*` sent to it. Agents call `https://<console host>/<platform>/<slug>/mcp`. With a
local engine (`AO_URL`) it needs `ONTOLOGY_PLATFORM` as well, or it refuses to start; the engine
runs from a checkout of NVIDIA's repository with `deploy/auto-ontology.override.yml`, on the host's
network as above. On Docker Desktop (macOS, Windows) host networking behaves differently: publish
the ports on loopback there instead. The rest of this document holds, with the console's host in
place of the platform's server.

## Firewall rules

### Inbound to a client's server

| From          | Port | Purpose                                                                                                                 | Notes                                                                                                                                                                                                                                                                                                                |
| ------------- | ---- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| internet      | 80   | the certificate's challenge, and the redirect to https                                                                  | Caddy obtains and renews the certificate itself                                                                                                                                                                                                                                                                      |
| internet      | 443  | the console at `/` (pages, `/api/host/*`, `/<platform>/<slug>/mcp`), the same under `/iblai`, NVIDIA's API under `/api` | admins' browsers, ibl.ai's agents on ibl.ai's egress and any console hosted elsewhere: none has an address list you can pin. Every `/api/host/*` call but `GET /api/host/health`, and every gateway call, needs an ibl.ai credential; NVIDIA's API takes its own keys. `/api/auth/*` and `/internal/*` answer `404`. |
| the operator  | 22   | SSH, and the tunnel to NVIDIA's own pages                                                                               |                                                                                                                                                                                                                                                                                                                      |
| anything else | —    | denied                                                                                                                  |                                                                                                                                                                                                                                                                                                                      |

### Inbound to one host on a VPN

| From                    | Port | Purpose                                           | Notes                                                                                                                                                                                                                                                              |
| ----------------------- | ---- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| VPN CIDR(s)             | 443  | console, gateway, playground                      | the only human entry point                                                                                                                                                                                                                                         |
| ibl.ai agent egress IPs | 443  | `/<platform>/<slug>/mcp` calls from ibl.ai agents | **Ask ibl.ai for the current egress list.** The A records below are ingress addresses and may not be the source of outbound agent traffic. Alternative: terminate ibl.ai traffic on the VPN concentrator or a private peering so no public inbound rule is needed. |
| anything else           | —    | denied                                            |                                                                                                                                                                                                                                                                    |

Ports 3000, 3001, 3002, 3100, 5434 and every Toolbox port are bound to loopback and are never
published.

### Outbound from the host

| To                                               | Port                                                                                          | Purpose                                                                    | Required by                                                                                                                                       |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ext.ibl.ai` → 129.80.99.227                     | 443                                                                                           | what a caller's credential is granted, checked on every call               | the server half's route handlers and gateway, at `ONTOLOGY_DM_URL` (else the build's `NEXT_PUBLIC_API_BASE_URL`)                                  |
| data source hosts                                | per engine (5432 PostgreSQL, 3306 MySQL, 443 Snowflake and HTTP APIs, ClickHouse's HTTP port) | schema reads, profiling, serving tools, fetching an API's OpenAPI document | auto-ontology, Toolbox, the server half (Fetch Spec)                                                                                              |
| `OPENAI_API_BASE_URL` (default `api.openai.com`) | 443                                                                                           | naming terms, resolving joins, embeddings, answering questions             | the console's model route, for the engine, and only with `OPENAI_API_KEY` set. Without it nothing goes out and the engine needs no model endpoint |
| the certificate authority                        | 443                                                                                           | obtaining and renewing the server's certificate                            | Caddy, on a client's server                                                                                                                       |
| DNS, NTP, internal CA / OS updates               | 53, 123, 443                                                                                  | hygiene                                                                    | host                                                                                                                                              |

When a client's server is installed or upgraded: GitHub (this repository), `mise.run`, `jdx.dev`
and `nodejs.org` (mise, and the Node it installs), the npm registry (`pnpm install` brings
Toolbox as `@toolbox-sdk/server`), Google Fonts (the console's typeface, fetched as it builds),
the container registries (`nvcr.io` for NVIDIA's engine, `docker.io` for Caddy and Postgres,
`public.ecr.aws` only for the console as an image) and Ubuntu's archive
([operations.md](/developer/guides/ontology-operations#what-the-server-needs)). On one host run by hand: the npm
registry. Mirror them internally if the host must stay closed after install, or set
`TOOLBOX_BIN` to a Toolbox binary you supply.

Roles, policies, API keys and agent connections are written on ibl.ai from the admin's browser,
not from the host.

Sign-in: the auth app only returns to origins on the platform's **allowed redirect origins** on
ibl.ai. Add every origin the console's pages are served from there before go-live:
`https://<host>` for a client's server, and the origin of any console hosted elsewhere or of one
host run by hand.

Checked on 2026-11-16: `ext.ibl.ai` does not answer from outside its allow-list (connections from an
unlisted address time out), while the hosted `api.iblai.app` does. Local development therefore uses
the hosted platform; the deployment sets `NEXT_PUBLIC_API_BASE_URL=https://ext.ibl.ai`.

Resolved on 2026-11-15: `ext.ibl.ai` = `129.80.99.227`; the hosted cloud (`api.iblai.app`,
`os.ibl.ai`, `login.iblai.app`) = `141.148.72.26`. Set `NEXT_PUBLIC_API_BASE_URL=https://ext.ibl.ai`
so every ibl.ai call from the browser and from the host uses the backend you allow-listed. A
`NEXT_PUBLIC_*` value is fixed when the console is built, so pages that call `ext.ibl.ai` come
from a build that had those addresses ([operations.md](/developer/guides/ontology-operations#another-iblai-deployment)).
The server
half takes its ibl.ai address at run time from `ONTOLOGY_DM_URL`. Pin by hostname at the proxy and
by IP at the firewall; re-check the IP when ibl.ai announces changes.

## What runs where

| Component           | Where                                                                                      | Listens                                                                         | State it keeps                                                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Pages               | the client's server (the console's process), a console hosted elsewhere, or one host       | the console's origin                                                            | none                                                                                                                                |
| Forwarding          | a console hosted elsewhere only                                                            | `/api/host/*`, sent on to the server in `x-ontology-server`                     | none                                                                                                                                |
| Server half         | the client's server (a systemd service, or a container on the host's network), or one host | 127.0.0.1:3000 behind the proxy; `/api/host/*`                                  | the files below; who a caller is kept in memory for 300 seconds, what they are granted for 60 seconds                               |
| Gateway             | same process                                                                               | `/<platform>/<slug>/mcp`, also under `/iblai` on a client's server              | `servers/<slug>/gateway.log`                                                                                                        |
| Model route         | same process                                                                               | `/internal/models/v1/*`, loopback only                                          | none                                                                                                                                |
| auto-ontology       | the client's server, as NVIDIA's own images; or beside one host                            | web app on 127.0.0.1:3100, its API on 3001, ingestion on 3002, Postgres on 5434 | its Postgres + pgvector volume (`postgres_data`)                                                                                    |
| Toolbox             | host, one process per server                                                               | 127.0.0.1, a free port recorded in `servers/<slug>/flags.json`                  | `servers/<slug>/{tools.yaml, flags.json, revisions/, pid, toolbox.log}`, per platform                                               |
| Generated secrets   | the client's server                                                                        | —                                                                               | the `secrets` volume, written once: `db-password`, `auth-secret`, `admin-password`, `model-key`                                     |
| Secrets for serving | host                                                                                       | —                                                                               | the platform's `env`, mode 0600: database passwords and API keys                                                                    |
| TLS proxy           | host                                                                                       | 0.0.0.0:443 (and :80 for Caddy)                                                 | Caddy's certificate, obtained automatically, in the `certificates` volume; on one host on a VPN, a certificate from the internal CA |

`ONTOLOGY_HOME` defaults to `.ontology` in the console's directory. On a client's server it is
`/var/lib/ontology`, or `/data` in the container (the `ontology_data` volume) when the console
runs as an image. Everything in it is kept per platform:

```
$ONTOLOGY_HOME/
  engine-key                      mode 0600: the key the server made for its own engine
  platforms/<platform>/
    engine.json                   a server without AO_URL: this platform's engine, {"url", "key"}
    env                           mode 0600: database passwords and API keys the runtime reads
    apis.yaml, actions.yaml
    servers/<slug>/               tools.yaml, flags.json, revisions/, pid, toolbox.log, gateway.log
```

The folder is the platform's key; a character outside `a-z 0-9 - _` is written as `~<hex>~`. Two
platforms can each have a server named `advising`.

## Identity on the wire

- Staff in the console: ibl.ai session tokens from `login` (see [auth.md](/developer/guides/ontology-auth)). The browser
  sends them to ibl.ai directly and to the console's route handlers, with the platform as
  `x-ontology-platform`; the handlers ask ibl.ai what they are granted on that platform. On a
  client's server the token reaches one place besides ibl.ai: the platform's own server. On a
  console hosted elsewhere the calls are forwarded with the same token, so it also passes through
  the hosted console.
- ibl.ai agents calling a server: the API key stored in the MCP server registered for that agent on
  ibl.ai, sent as `Authorization: Api-Token …`. It must belong to the platform in the server's
  address.
- People using Claude, Codex or Cursor: an API key issued from the console, as `Api-Token` or
  `Bearer`. When the server declares an OIDC auth service, the ID token rides along and Toolbox
  verifies it.
- The console holds no ibl.ai secret and issues no tokens, wherever it runs. The server checks each
  credential by asking ibl.ai with that credential.
- The engine: the server's own engine key, sent only to `127.0.0.1:3100`. The engine's model calls
  carry its generated model key, sent only to the console on loopback.

## Hardening checklist

- A client's server has a public DNS name, set as `ONTOLOGY_HOST`, and Caddy obtains its
  certificate. One host on a VPN joins no public DNS; its proxy serves an internal-CA certificate.
- The proxy sets `X-Forwarded-Proto: https`. The gateway refuses a credential that arrived over
  plain HTTP from anywhere but loopback.
- A server whose `ONTOLOGY_HOST` is `localhost` answers on its own machine only: it takes an HTTP
  API on that machine, which a server with a public host name refuses, and any
  `http://localhost:<port>` origin.
- Each `$ONTOLOGY_HOME/platforms/<platform>/env` holds database passwords and API keys, mode 0600,
  owned by the user the server half runs as. The engine can keep its own connection credentials in
  Vault if you already run one. Database passwords stay on the platform's server; a console hosted
  elsewhere holds none.
- The four generated secrets are in the `secrets` volume, never in `compose.yaml` or a `.env`, and
  the server's engine key is in `$ONTOLOGY_HOME/engine-key`. `OPENAI_API_KEY`, when set, is the
  one secret the operator supplies: keep the file that holds it closed to other users (mode 600,
  as the install leaves `.env`).
- NVIDIA's own pages are reachable only through the SSH tunnel, behind their admin password. Its
  API under `/api` takes its own keys; `/api/auth/*` and `/internal/*` answer `404`.
- Database logins are read-only roles (see [guides/connect-data-sources.md](/developer/guides/ontology-connect-data-sources)).
  Write tools use a separate login and a separate toolset.
- Allowed origins and hosts are set per server. Toolbox receives both as flags but does not refuse
  a foreign `Origin`, so the gateway checks both before it checks the credential.
- NVIDIA's services are its own published images at one release; pin the images Compose pulls by
  digest if your policy asks for it. Toolbox is pinned by the lockfile, on one host and on a client's server
  alike.
- Logs: the proxy's access log, `gateway.log` and `toolbox.log` per server, the engine's service
  logs. Ship them to your SIEM from the host.
