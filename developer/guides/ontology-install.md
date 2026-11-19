# Install ibl.ai/ontology

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/install.md`](https://github.com/iblai/ontology/blob/main/docs/install.md). This page is generated — edit it in the repository, not here.

Four blocks to paste on a fresh Ubuntu server, and one step on ibl.ai. Most of the time goes to
one build and one download.

What ends up on the server:

- **ibl.ai/ontology**, built there from this repository's source and run by systemd. Nothing of
  ibl.ai's arrives as a prebuilt image.
- **NVIDIA's auto-ontology engine, its database and Caddy**, in Docker. The engine is NVIDIA's
  own images, pulled from its registry, and Caddy gets the HTTPS certificate.

## Before you start

- **An Ubuntu server**: 22.04, 24.04 or 26.04 on x86-64, with 4 GB of memory and 15 GB free, and
  a user who can `sudo`. A fresh one is best.
- **A host name** that points to the server, such as `ontology.example.edu`, with ports 80 and
  443 open to the internet. Create the DNS record first: the certificate cannot be issued until
  the name resolves to the server.
- **Your ibl.ai platform key**, such as `example-university`, and an administrator of that
  platform. Check [login.iblai.app/me](https://login.iblai.app/me) for this info.

Paste each block into a terminal on the server and press Enter. Only the first one needs your
values.

## 1. Name the platform and the host

Replace `example-university` and `ontology.example.edu` with yours, and allow 80 and 443 ports via `ufw`. Run this block once: a second
run writes the file anew.

```bash
sudo mkdir -p /opt/iblai-ontology
sudo tee /opt/iblai-ontology/.env <<'EOF'
ONTOLOGY_PLATFORM=example-university
ONTOLOGY_HOST=ontology.example.edu
EOF
sudo chmod 600 /opt/iblai-ontology/.env
command -v ufw >/dev/null 2>&1 && {
    sudo ufw allow 80
    sudo ufw allow 443
}
```

## 2. Install Docker

NVIDIA's engine, its database and Caddy run in it. The last two lines make `ontology`, the user
that builds and runs ibl.ai/ontology, and give it `/opt/iblai-ontology` to build in.

```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-v2 git curl libatomic1
sudo useradd --system --create-home -s /bin/bash ontology
sudo chown -R ontology: /opt/iblai-ontology
```

## 3. Build ibl.ai/ontology

[mise](https://mise.jdx.dev) gives the `ontology` user the Node and pnpm versions this repository
names, and pnpm builds it from the source.

```bash
cd /opt/iblai-ontology
sudo -u ontology -H bash -eo pipefail -c '
  export NEXT_TELEMETRY_DISABLED=1 PATH="$HOME/.local/bin:$HOME/.local/share/mise/shims:$PATH"
  echo '\''export PATH="$HOME/.local/bin:$PATH"'\'' >> $HOME/.bashrc
  echo '\''export PATH="$HOME/.local/share/mise/shims:$PATH"'\'' >> $HOME/.bashrc
  command -v mise >/dev/null || curl -fsSL https://mise.run | MISE_INSTALL_HELP=0 sh
  test -d repo || git clone --branch main https://github.com/iblai/ontology.git repo
  cd repo
  mise install
  pnpm install --frozen-lockfile
  pnpm build
'
```

## 4. Start everything

```bash
cd /opt/iblai-ontology
sudo curl -fsSLO https://raw.githubusercontent.com/iblai/ontology/main/deploy/compose.yaml
sudo curl -fsSLo /etc/systemd/system/iblai-ontology.service https://raw.githubusercontent.com/iblai/ontology/main/deploy/iblai-ontology.service
sudo docker compose up -d
sudo systemctl daemon-reload
sudo systemctl enable --now iblai-ontology
curl -fsS --retry 30 --retry-delay 1 --retry-connrefused http://127.0.0.1:3000/api/host/health; echo
```

`docker compose up -d` downloads NVIDIA's engine the first time, about half a gigabyte. The
last line prints `{"ok":true}` once ibl.ai/ontology is up.

## 5. Let people sign in

Open `https://<your host>/` and sign in with your ibl.ai account on that platform.

## If a step fails

Blocks 2, 3 and 4 can be pasted again as they are: each takes up where it stopped.

- **Block 2 says `Could not get lock`**: Ubuntu is updating itself. Wait a minute and paste it
  again.
- **Block 4 stops with `DeadlineExceeded` at `nvcr.io`**: NVIDIA's registry timed out. Paste it
  again.
- **The last line prints no `{"ok":true}`**: `sudo journalctl -u iblai-ontology -n 30` says why
  it did not start.
