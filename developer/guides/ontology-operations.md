# Operate ibl.ai/ontology

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/operations.md`](https://github.com/iblai/ontology/blob/main/docs/operations.md). This page is generated — edit it in the repository, not here.

For the IT team that looks after a server installed as [install.md](/developer/guides/ontology-install) says: what runs
on it, its settings, giving the engine a model, upgrades, backups, and what to do when something
stops. The last sections cover running ibl.ai/ontology in Docker as an image, and a console
hosted elsewhere. On this page "the console" is ibl.ai/ontology itself, next to NVIDIA's
engine.

## What runs on the server

| Part                          | How it runs                                                                                                 | What it does                                                                                       |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| ibl.ai/ontology, the console  | A systemd service, `iblai-ontology`, as the user `ontology`, from the source in `/opt/iblai-ontology/repo`. | The pages you manage it from, the API those pages call, and the MCP servers your agents call.      |
| NVIDIA's auto-ontology engine | Docker: its web app, its API and its ingestion service, as NVIDIA's own images at one release.              | It reads your systems and builds the terms.                                                        |
| The engine's database         | Docker: Postgres, for the engine alone.                                                                     | Keeps the systems the engine reads, its terms and what was certified, its users and keys.          |
| Caddy                         | Docker.                                                                                                     | Gets and renews the certificate for your host name, sends http on to https, and is the one way in. |

What the internet reaches, all through Caddy on ports 80 and 443: the console, the MCP servers,
and NVIDIA's API. What it does not: NVIDIA's sign-in and its own pages, and the engine's model
route. The console (3000), NVIDIA's web app (3100), its API (3001), its ingestion service (3002),
the database (5434) and every MCP server's process listen on the server's loopback only.

Your ibl.ai platform keeps what it keeps today: the sign-ins, the roles, and one key per agent you
connect. The server serves that one platform and no other.

Where things are:

| Path                                            | What                                                                                                   |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `/opt/iblai-ontology/.env`                      | The settings, for Docker and for the console alike.                                                    |
| `/opt/iblai-ontology/compose.yaml`              | What Docker runs: the repository's `deploy/compose.yaml`, downloaded by root.                          |
| `/opt/iblai-ontology/repo`                      | The console's source and its build. `/home/ontology/.local` holds mise, Node and pnpm.                 |
| `/etc/systemd/system/iblai-ontology.service`    | How the console is started: `deploy/iblai-ontology.service`, downloaded by root.                       |
| `/var/lib/ontology`                             | The console's files: the MCP servers deployed on it, their revisions, the logs, its key to the engine. |
| Docker volumes named `iblai-ontology-console_…` | `secrets`, `postgres_data` (the engine's database) and `certificates` (Caddy's).                       |

## What the server needs

[install.md](/developer/guides/ontology-install) has the short list. In full:

- **Ubuntu 22.04, 24.04 or 26.04 on x86-64**, with 4 GB of memory and 15 GB free where Docker
  keeps its data: the console is built on the server, and the engine's images and data live there.
- **Nothing else listening on 80, 443, 3000, 3001, 3002, 3100 or 5434**: the services run on the
  server's own network.
- **No older ibl.ai ontology backend on it.** A server that runs v0.2.x keeps its files in
  `/opt/iblai-ontology` and holds 80 and 443. Remove it first with its own installer (see
  [When something stops](#when-something-stops)).
- **Docker with Compose 2.23.1 or later, and git** for the console's source. Ubuntu's own
  archive has all three. Nothing of NVIDIA's is built on the server: its images are pulled.
- **Outbound access while it is installed or upgraded** to GitHub (with
  `raw.githubusercontent.com`), `mise.run`, `jdx.dev`, `nodejs.org`, the npm registry, Google
  Fonts (`fonts.googleapis.com`, `fonts.gstatic.com`), `nvcr.io` (NVIDIA's images), Docker Hub
  (Postgres and Caddy) and Ubuntu's archive: the console is built from them and the rest pulled.
  **Once it runs**, to your ibl.ai platform, to the data sources you connect, to Let's Encrypt
  to renew the certificate, and to your model's address if you give it one.

`/opt/iblai-ontology` belongs to the `ontology` user, which builds in it. The console's unit is
downloaded from GitHub by root into `/etc/systemd/system` and is root's alone. `.env` and
`compose.yaml` sit in that user's directory, so the service can change them as well as root can.

## The settings

They live in `/opt/iblai-ontology/.env`. Docker Compose reads the file for the engine and Caddy,
and the console's service reads the same file. Write one `NAME=value` per line, with no quotes
and nothing after the value; a comment is a line of its own that starts with `#`. Set them
there and not at the top of `compose.yaml`, which the console does not read.

| Setting                  | Default                         | What it does                                                                                            |
| ------------------------ | ------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `ONTOLOGY_PLATFORM`      | None: nothing starts without it | Your platform's key. The server serves that platform only; another platform's administrator is refused. |
| `ONTOLOGY_HOST`          | `localhost`                     | The host name. Caddy gets the certificate for it.                                                       |
| `ONTOLOGY_DM_URL`        | `https://api.iblai.app/dm`      | The ibl.ai platform's address, where every sign-in and key is checked.                                  |
| `OPENAI_API_KEY`         | Empty                           | Turns the semantic features on (see [A model](#a-model)).                                               |
| `OPENAI_API_BASE_URL`    | `https://api.openai.com/v1`     | Where the model calls go: OpenAI, or a gateway that speaks its API.                                     |
| `OPENAI_MODEL`           | `gpt-4.1`                       | The chat model.                                                                                         |
| `OPENAI_EMBEDDING_MODEL` | `text-embedding-3-large`        | The embedding model.                                                                                    |
| `ONTOLOGY_LOCAL_ROOT`    | See below                       | The one folder a Filesystem source may read. Nothing outside it is listed or read.                      |

Without `ONTOLOGY_HOST` the console answers at `https://localhost`, from the server itself only,
with a certificate of Caddy's own that browsers warn about.

`ONTOLOGY_LOCAL_ROOT` defaults to `/home/ontology` when the console runs from source, and the
files must be readable by the `ontology` user. In Docker it defaults to `/opt/iblai-ontology/files`,
mounted read-only into the console at the same path; Docker makes the folder, empty, if it is not
there, and its files must be readable by uid 1000, the image's `node` account.

After a change, both read the file again:

```bash
cd /opt/iblai-ontology
sudo docker compose up -d
sudo systemctl restart iblai-ontology
```

`ONTOLOGY_PLATFORM` is the one setting not to change on a server that holds data. The engine's
connected systems and terms belong to the platform it was installed for, and a server pointed at
another platform would serve them to it. To serve another platform, remove the install with its
data (see [Day to day](#day-to-day)) and install again.

## Addresses

| Address                                                      | What                                                                                                       |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `https://<host>/`                                            | The console.                                                                                               |
| `https://<host>/<platform>/<server>/mcp`                     | An MCP server you deploy in the console. Agents call it with their own key.                                |
| `https://<host>/api/host/...`                                | The API the console's pages call, with your sign-in. `/api/host/health` answers anyone with `{"ok":true}`. |
| `https://<host>/api/...`                                     | NVIDIA's API, with the engine's own keys: everything under `/api` except `/api/host` and `/api/auth`.      |
| `https://<host>/api/auth/...`, `https://<host>/internal/...` | 404. NVIDIA's sign-in and the engine's model route stay on the server.                                     |
| `https://<host>/iblai/...`                                   | A second address for a console hosted elsewhere (see the last section).                                    |

Sign-in returns only to an address the platform allows: `https://<host>` must be among the
platform's allowed redirect origins on ibl.ai. Without it, signing in ends on ibl.ai with the
address refused.

## A model

Without a model, connecting a database, the semantic map, certifying terms, and the raw SQL and
certified-term tools of an MCP server all work; the engine names its terms from the table and
column names. Joins it would infer, and answering questions in plain words, need a model.

To give it one, add the key to `.env` and restart the console:

```bash
echo 'OPENAI_API_KEY=<your key>' | sudo tee -a /opt/iblai-ontology/.env >/dev/null
sudo systemctl restart iblai-ontology
```

The engine never holds the key. It calls a route of the console on the server's loopback with a
generated key of its own, and the console adds your key and sends the call on to
`OPENAI_API_BASE_URL`, with `OPENAI_MODEL` for chat and `OPENAI_EMBEDDING_MODEL` for embeddings.

- **A gateway** that speaks OpenAI's API goes in `OPENAI_API_BASE_URL`. One that names its models
  `provider/model`, such as the ibl.ai DM or OpenRouter, takes that id in `OPENAI_MODEL`. The DM has
  no embeddings route yet, so with it only chat works for now.
- **The chat model** must take an output-token limit as a chat model does. The engine sets small
  limits, which a reasoning model's hidden tokens would use up.
- **The embedding model** must be able to answer in 2048 dimensions. `text-embedding-3-large` can;
  `text-embedding-3-small` cannot.

## The generated secrets

Nothing secret is typed, and none is kept in `compose.yaml`. On the first start a one-shot
service, `secrets`, writes four random secrets into the `secrets` volume, and never again:

| Secret           | What it is                                                  |
| ---------------- | ----------------------------------------------------------- |
| `db-password`    | The engine's database password.                             |
| `auth-secret`    | Signs the engine's sign-in sessions.                        |
| `admin-password` | The password of the engine's admin, `admin@ontology.local`. |
| `model-key`      | The engine's key to the console's model route.              |

The console reads two of them, `admin-password` and `model-key`. systemd hands them to its
service from that volume each time it starts (`LoadCredential=` in the unit), so there is no
second copy to keep in step. The unit names Docker's default place for volumes,
`/var/lib/docker/volumes`. A console that is already running keeps the ones it started with:
restart it after the volume is replaced.

The console makes its own key to the engine: it signs in as the engine's admin, makes a key and
keeps it in `/var/lib/ontology`. Nobody mints a key by hand.

The `secrets` and `postgres_data` volumes go together: the database was created with that
password. Back them up together, and never delete one without the other.

To read the engine admin's password:

```bash
cd /opt/iblai-ontology
sudo docker compose exec postgres cat /secrets/admin-password
```

NVIDIA's own pages are not served on the internet: on the same host name they would share an
origin with the console and could read its sign-in. To open them, tunnel from your computer:

```bash
ssh -L 3100:127.0.0.1:3100 <you>@<host>
```

Then open `http://localhost:3100` and sign in as `admin@ontology.local` with that password.

## Day to day

```bash
systemctl status iblai-ontology                             # is the console running
sudo journalctl -u iblai-ontology --follow                  # the console's log
sudo systemctl restart iblai-ontology                       # restart the console

cd /opt/iblai-ontology
sudo docker compose ps                                      # the engine, its database, Caddy
sudo docker compose logs --tail 50 auto-ontology-frontend   # NVIDIA's web app
sudo docker compose restart                                 # restart all three
```

**Upgrade**: stop the console and take the newest source, then paste blocks 3 and 4 of
[install.md](/developer/guides/ontology-install) again.

```bash
cd /opt/iblai-ontology
sudo systemctl stop iblai-ontology
sudo -u ontology -H git -C /opt/iblai-ontology/repo fetch origin main
sudo -u ontology -H git -C /opt/iblai-ontology/repo reset --hard origin/main
```

Block 3 builds the new source; block 4 takes the newest `compose.yaml` and unit and starts
everything. The console is down while it builds, a minute or two: left running, it would answer
from a build that is being replaced, and block 4 would not restart it. `git reset --hard` puts
the tracked files as `main` has them; the build, `node_modules` and a `.env.local` of yours
stay. When the new `compose.yaml` names another release of NVIDIA's engine, block 4 pulls it and
the engine brings its database up to date as it starts. If the build fails, the console stays stopped: fix what it
names and paste the block again.

**Back up**: `/opt/iblai-ontology/.env`, `/var/lib/ontology`, and the two Docker volumes that go
together. Stop first, so the database and the files are copied whole:
`sudo systemctl stop iblai-ontology` and `sudo docker compose stop`.

| What                                          | What it holds                                                                                          |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `/var/lib/ontology`                           | The console's files: the MCP servers deployed on it, their revisions, the logs, its key to the engine. |
| Volume `iblai-ontology-console_secrets`       | The four generated secrets. Goes with `postgres_data`.                                                 |
| Volume `iblai-ontology-console_postgres_data` | The engine's database: the systems it reads, its terms and what was certified, its users and keys.     |
| Volume `iblai-ontology-console_certificates`  | Caddy's certificate. It is issued again by itself, so it needs no backup.                              |

**Remove**: remove each agent's connection in the console first, so its key is deleted in your
platform. Then:

```bash
sudo systemctl disable --now iblai-ontology
sudo rm /etc/systemd/system/iblai-ontology.service
sudo systemctl daemon-reload
cd /opt/iblai-ontology && sudo docker compose down     # the containers go; the data stays
```

And to remove the data, the settings and the console's user as well:

```bash
cd /opt/iblai-ontology && sudo docker compose down --volumes
cd / && sudo rm -rf /opt/iblai-ontology /var/lib/ontology
sudo userdel --remove ontology
```

## Another ibl.ai deployment

The settings above point the server at the hosted ibl.ai platform. For another ibl.ai deployment,
`ONTOLOGY_DM_URL` is the server's half of the choice. The browser's half is fixed when the
console is built: write that deployment's addresses into `.env.local` in the checkout, then
build again as an upgrade does.

```bash
sudo -u ontology tee /opt/iblai-ontology/repo/.env.local <<'EOF'
NEXT_PUBLIC_API_BASE_URL=<...>
NEXT_PUBLIC_AUTH_URL=<...>
NEXT_PUBLIC_BASE_WS_URL=<...>
NEXT_PUBLIC_PLATFORM_BASE_DOMAIN=<...>
NEXT_PUBLIC_LEGACY_LMS_URL=<...>
NEXT_PUBLIC_MFE_URL=<...>
EOF
```

Leave out a line rather than leave it empty: a name that is not there keeps the hosted
platform's address, an empty one blanks it.

## The console as an image

Building the console on the server is the default. Docker can instead run it as an image:
ibl.ai's prebuilt one, `public.ecr.aws/ibleducation/iblai-ontology`, or one
[you build on the server](#or-build-the-image-yourself). `COMPOSE_PROFILES=all-in-one` in `.env`
turns that on, and then no `ontology` user, mise or service is needed. Use one or the other on a
server, never both: they hold the same port.

ibl.ai's image, on a server with Docker and Compose 2.23.1 or later, and outbound access to
`public.ecr.aws` as well:

```bash
sudo mkdir -p /opt/iblai-ontology && cd /opt/iblai-ontology
sudo curl -fsSLO https://github.com/iblai/ontology/releases/latest/download/compose.yaml
sudo tee .env <<'EOF'
ONTOLOGY_PLATFORM=acme-university
ONTOLOGY_HOST=ontology.example.edu
COMPOSE_PROFILES=all-in-one
EOF
sudo chmod 600 .env
sudo docker compose up -d
```

What differs from the rest of this page:

- **The console's files** are in the Docker volume `iblai-ontology-console_ontology_data`, not
  in `/var/lib/ontology`. Back it up with the other two.
- **Its log**: `sudo docker compose logs --follow ontology`. `sudo docker compose ps` shows it
  with the others.
- **After a change to `.env`**: `sudo docker compose up -d` alone.
- **Upgrade**: download the new `compose.yaml` over the old one and `sudo docker compose up -d`.
  The console's new image is pulled, and NVIDIA's when the release names another.
- **Remove**: `sudo docker compose down`, with `--volumes` to remove the data too.
- **Another ibl.ai deployment** builds its own image from this repository, at the release's
  tag, with its addresses as build arguments, and names that image on the `image:` line of the
  `ontology` service in `compose.yaml`:

  ```bash
  git clone --branch v<version> https://github.com/iblai/ontology
  cd ontology
  docker build -t <registry>/ontology:<version> \
    --build-arg NEXT_PUBLIC_API_BASE_URL=<...> \
    --build-arg NEXT_PUBLIC_AUTH_URL=<...> \
    --build-arg NEXT_PUBLIC_BASE_WS_URL=<...> \
    --build-arg NEXT_PUBLIC_PLATFORM_BASE_DOMAIN=<...> \
    --build-arg NEXT_PUBLIC_LEGACY_LMS_URL=<...> \
    --build-arg NEXT_PUBLIC_MFE_URL=<...> \
    .
  ```

### Or build the image yourself

Nothing of ibl.ai's is pulled: the image is built on the server from this repository's
[`Dockerfile`](https://github.com/iblai/ontology/blob/main/Dockerfile), under the name `compose.yaml` asks for, so Compose finds it there.
[install_docker_local.md](https://github.com/iblai/ontology/blob/main/docs/install_docker_local.md) is that install: four blocks to paste, and
Docker is all the server needs.

What differs from the rest of this page is as listed above, but for the upgrade:
`sudo git -C /opt/iblai-ontology/repo pull`, then blocks 3 and 4 of that guide again. Leave
`docker compose pull` out on this route: it asks ibl.ai's registry for that name. For another
ibl.ai deployment, give `docker build` that deployment's addresses as `--build-arg`, as above.

On a machine of your own it runs from a checkout ([dev_local.md](https://github.com/iblai/ontology/blob/main/docs/dev_local.md)): `git pull`
there, then its build and start blocks again.

### Or let the script do it

On a fresh Ubuntu server, one command installs the console as an image:

```bash
curl -fsSL https://github.com/iblai/ontology/releases/latest/download/install.sh \
  | sudo bash -s -- --host ontology.example.edu --platform acme-university
```

To read the script before running it:
`curl -fsSL https://github.com/iblai/ontology/releases/latest/download/install.sh | less`.

It:

1. **Checks the server** before it changes anything: root, Ubuntu 22.04 or later, x86-64, `curl`
   and `ss`, no older ibl.ai ontology backend in `/opt/iblai-ontology`, no console built from
   source, and on a first install the ports above free and 15 GB of room and 4 GB of memory.
   Otherwise it stops and says why.
2. **Docker**: uses the one that is there, with Compose 2.23.1 or later. With no Docker it
   installs `docker.io` and `docker-compose-v2` from Ubuntu's own archive; no repository or key
   file is added.
3. **Asks for the platform's key** if the command did not carry it.
4. **Writes `/opt/iblai-ontology/.env`** with the names it was given (`ONTOLOGY_PLATFORM`,
   `ONTOLOGY_HOST`, `ONTOLOGY_DM_URL`), `COMPOSE_PROFILES=all-in-one`, and no secret. Lines you
   added, such as `OPENAI_API_KEY`, are left as they are.
5. **Downloads** the `compose.yaml` of its own release into `/opt/iblai-ontology`.
6. **Pulls the images**, up to three times, since NVIDIA's registry sometimes times out.
7. **Starts everything** and waits up to five minutes for `https://<host>/api/host/health`.
8. **Prints** the addresses, the redirect-origin step, how to read the engine admin's password and
   open NVIDIA's pages, and the day-to-day commands. It never prints a secret.

Run it again to upgrade: `.env` keeps what was given, and `--host` or `--platform-url` given again
replaces it. A `--platform` other than the one the server already serves is refused.

| Option                     |                                                                                                                |
| -------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `--platform <key>`         | The platform this server serves. Asked for if not given.                                                       |
| `--host <host name>`       | The name the server answers at. Without it, `localhost`.                                                       |
| `--platform-url <address>` | The ibl.ai platform's address, when it is not `https://api.iblai.app/dm`.                                      |
| `--dry-run`                | Say what would happen; change nothing and ask nothing.                                                         |
| `--uninstall`              | Remove the console, its data and `/opt/iblai-ontology`. It asks you to type the host name first. Docker stays. |
| `--help`                   | Print this list.                                                                                               |

## When something stops

Two places say why: `sudo journalctl -u iblai-ontology -n 30` for the console, and
`sudo docker compose logs --tail 30 <service>` in `/opt/iblai-ontology` for the rest.

| What you see                                                                                                                                | What it means                                                                                                       | What to do                                                                                                                                                                                                                                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Compose stops with `required variable ONTOLOGY_PLATFORM is missing a value`                                                                 | The platform is not set.                                                                                            | Set `ONTOLOGY_PLATFORM` in `.env` (see [The settings](#the-settings)).                                                                                                                                                                                                                              |
| The console's log says `ONTOLOGY_PLATFORM is not set`, and the service starts again every five seconds                                      | The same, seen by the console.                                                                                      | Set it in `.env`. The service comes up by itself at its next try.                                                                                                                                                                                                                                   |
| The console's log says `Failed to set up credentials: Protocol error`, and `systemctl status iblai-ontology` shows `status=243/CREDENTIALS` | The engine's secrets are not where the unit looks: Docker has not written them yet, or keeps its volumes elsewhere. | Run `sudo docker compose up -d` first: it writes them as it starts the services, and the console comes up at its next try. If Docker's data is not in `/var/lib/docker`, put the path `sudo docker volume inspect iblai-ontology-console_secrets` prints in the unit's two `LoadCredential=` lines. |
| `systemctl status iblai-ontology` shows `status=203/EXEC`                                                                                   | mise is not in `/home/ontology/.local/bin`: block 3 of the install did not finish.                                  | Paste block 3 again.                                                                                                                                                                                                                                                                                |
| `apt-get` says `Could not get lock`                                                                                                         | Ubuntu is updating itself, as it does on a server's first minutes.                                                  | Wait a minute and run it again.                                                                                                                                                                                                                                                                     |
| `https://<host>/` answers 502                                                                                                               | Caddy is up and the console is not.                                                                                 | Read the console's log.                                                                                                                                                                                                                                                                             |
| `pnpm build` ends with `Killed`                                                                                                             | The server ran out of memory while building.                                                                        | Give it 4 GB, or add swap, and build again.                                                                                                                                                                                                                                                         |
| `sudo docker compose up -d` stops at `nvcr.io` with `DeadlineExceeded`                                                                      | Pulling NVIDIA's images timed out. It reads like a missing login; it is not: they are public.                       | `sudo docker compose up -d` again.                                                                                                                                                                                                                                                                  |
| Node says `error while loading shared libraries: libatomic.so.1`                                                                            | The server lacks a library Node's binary needs, which block 2 of the install brings.                                | `sudo apt-get install -y libatomic1`, then paste block 3 again.                                                                                                                                                                                                                                     |
| `proxy` keeps restarting; `sudo docker compose logs proxy` says `address already in use`                                                    | Something else holds 80 or 443, such as the older ibl.ai ontology backend (v0.2.x).                                 | Stop it. The older backend is removed with its own installer: `curl -fsSL https://github.com/iblai/ontology/releases/download/v0.2.13/install.sh \| sudo bash -s -- --uninstall`.                                                                                                                   |
| The browser warns about the certificate, and `sudo docker compose logs proxy` shows the certificate failing                                 | Let's Encrypt cannot reach the server by its host name.                                                             | Check that `ONTOLOGY_HOST` points to this server and that 80 and 443 are open to the internet. Caddy tries again by itself.                                                                                                                                                                         |
| Signing in ends on ibl.ai with the address refused                                                                                          | `https://<host>` is not among the platform's allowed redirect origins.                                              | Add it (see [Addresses](#addresses)).                                                                                                                                                                                                                                                               |
| An administrator gets 403 `This server serves another platform.`                                                                            | They signed in to another platform than `ONTOLOGY_PLATFORM`.                                                        | Sign in to the platform this server serves.                                                                                                                                                                                                                                                         |

## A console hosted elsewhere

The console's pages can also run somewhere else, built with `NEXT_PUBLIC_ONTOLOGY_RELAY=1`, and
reach this server as theirs. They call it at `https://<host>/iblai/...`: `/iblai/api/host/...` and
`/iblai/<platform>/<server>/mcp`, which Caddy sends on to the console with `/iblai` taken off. The
server is installed the same way; nothing changes on it.
