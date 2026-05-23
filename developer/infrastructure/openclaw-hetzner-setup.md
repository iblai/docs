# OpenClaw on Hetzner — Setup Guide

Step-by-step instructions for deploying an OpenClaw gateway on a Hetzner Cloud server and connecting it to ibl-dm-pro as a chat runner.

---

## Architecture

```
Student (browser) → ibl-dm-pro (Django Channels / ASGI)
                         │
                    ClawLLMRunner
                         │
                    OpenClawClient (WSS + Ed25519 device identity signing)
                         │
                    Caddy (on host, TLS via Let's Encrypt)
                         │ reverse proxy to localhost:18789
                         ▼
                    OpenClaw Gateway (systemd service, loopback only)
                         │
                    LLM Provider (Anthropic, etc.)
```

**Why Caddy on the host (not Docker)**: Caddy must run directly on the host so that TCP connections to OpenClaw arrive from `127.0.0.1`. This preserves loopback auto-approval for device identity — the same mechanism that made moltworker work without explicit device signing on Cloudflare. If Caddy ran in a Docker container, it would connect via Docker bridge (172.x.x.x) and OpenClaw would treat it as a remote connection.

**Why device identity signing**: On vanilla OpenClaw (unlike moltworker), the gateway requires Ed25519 device identity in the WebSocket connect handshake. Without it, connections succeed but the gateway grants **zero scopes** — effectively treating the client as unauthenticated. This was the root cause of the "missing scope: operator.read" failure encountered during initial deployments. The DM backend now signs each connect with its own Ed25519 keypair.

---

## Prerequisites

Before starting, you need:

1. **Hetzner Cloud server** — CX22 (2 vCPU, 4 GB RAM, €3.49/mo) is sufficient. OpenClaw is lightweight; the LLM API call is the bottleneck, not local compute. Use the Ashburn location for US East proximity.
2. **A domain or subdomain** pointing to the server's **actual IP** (not an elastic IP — see Snag section below).
3. **Anthropic API key** (or another LLM provider key).
4. **Ports 80 and 443 open** on the Hetzner Cloud Firewall **before** installing Caddy.
5. **Admin access** to the ibl-dm-pro Django admin.

### Critical: DNS and firewall must be ready first

Let's Encrypt ACME challenges will fail if:
1. DNS points to an elastic IP that isn't routing to the actual server
2. Port 443 is not open on the Hetzner Cloud Firewall (only port 80 was initially opened)

After 5 failed attempts, Let's Encrypt rate-limits the domain for 1 hour. **All three** of these must be correct before Caddy's first start:
- DNS A record → server's real IP (verify with `dig your-domain.example.com +short`)
- Port 80 open inbound from `0.0.0.0/0` (for `http-01` ACME challenge)
- Port 443 open inbound (for `tls-alpn-01` fallback and actual HTTPS traffic)

Also: don't toggle firewall rules while Caddy is retrying — each failed attempt counts against the rate limit.

---

## Part 1: Install OpenClaw

### 1.1 — SSH in and install Node.js 22

```bash
ssh root@<server-ip>

# Install Node.js 22 (skip if already installed — check with node --version)
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs
node --version  # should show v22.x.x
```

### 1.2 — Install OpenClaw

```bash
npm install -g openclaw@latest
openclaw --version
```

### 1.3 — Generate a gateway token

```bash
export OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)
echo "$OPENCLAW_GATEWAY_TOKEN"
```

**Save this token** — you need it for the Django `ClawInstance` record later.

Immediately persist it to `~/.bashrc` so CLI commands work in future SSH sessions. Running `openclaw devices list` in a new SSH session will fail with `MissingEnvVarError: Missing env var "OPENCLAW_GATEWAY_TOKEN"` if the token was only exported in the original shell:

```bash
echo "export OPENCLAW_GATEWAY_TOKEN=$OPENCLAW_GATEWAY_TOKEN" >> ~/.bashrc
```

### 1.4 — Write the full config

Writing the full config upfront skips the interactive onboarding wizard entirely.

```bash
mkdir -p ~/.openclaw

cat > ~/.openclaw/openclaw.json << 'CONF'
{
  "meta": {
    "lastTouchedVersion": "<your-installed-version>"
  },
  "wizard": {
    "lastRunVersion": "<your-installed-version>",
    "lastRunCommand": "onboard",
    "lastRunMode": "local"
  },
  "auth": {
    "profiles": {
      "anthropic:default": {
        "provider": "anthropic",
        "mode": "api_key"
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6"
      },
      "workspace": "/root/.openclaw/workspace"
    }
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "session": {
    "dmScope": "per-channel-peer"
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "controlUi": {
      "allowedOrigins": [
        "https://your-domain.example.com"
      ]
    },
    "auth": {
      "mode": "token",
      "token": "${OPENCLAW_GATEWAY_TOKEN}"
    },
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    }
  }
}
CONF
```

Replace `<your-installed-version>` with the output of `openclaw --version` (e.g. `2026.3.13`). Replace `https://your-domain.example.com` in `controlUi.allowedOrigins` with your actual domain. Change the model in `agents.defaults.model.primary` if needed (OpenClaw normalizes date-stamped IDs to short aliases, e.g. `claude-sonnet-4-20250514` → `claude-sonnet-4-6`).

The `wizard` and `meta` fields tell OpenClaw that onboarding already ran, so `openclaw onboard` won't re-prompt. The `session.dmScope: "per-channel-peer"` is a security best practice for multi-user (each DM conversation gets its own session scope).

**Optional: model fallbacks** — to prevent hard failures when the primary LLM provider has an outage, you can add fallback models:

```json
"model": {
    "primary": "anthropic/claude-sonnet-4-6",
    "fallbacks": ["anthropic/claude-haiku-4-5", "openai/gpt-5"]
}
```

This is especially recommended for multi-agent setups where the probability of hitting an API error scales with the number of agents.

### 1.5 — Set the Anthropic API key

```bash
export ANTHROPIC_API_KEY=<your-key>
echo "export ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY" >> ~/.bashrc
```

### 1.6 — Create systemd service and start

```bash
# Create workspace directory
mkdir -p /root/.openclaw/workspace

# Enable lingering so user services survive SSH logout
loginctl enable-linger root

# Start the gateway
openclaw gateway --port 18789 &

# Or if you prefer, run the onboard wizard just for the systemd service:
# openclaw onboard --install-daemon
# (It will detect existing config and skip most prompts)
```

Verify the gateway is running:

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
# Expected: 200
```

> **Note**: If you want the systemd service auto-created, you can still run `openclaw onboard --install-daemon` — it will detect the existing config ("Use existing values"), skip most prompts, and just install the service at `~/.config/systemd/user/openclaw-gateway.service`.

> **Why `loginctl enable-linger root`?** OpenClaw installs a **user-level** systemd service. Without lingering, the service dies when the last SSH session closes. The `openclaw onboard --install-daemon` wizard handles this automatically, but if you skip the wizard you must run it yourself. Verify with: `loginctl show-user root 2>/dev/null | grep Linger` — should show `Linger=yes`. See Snag #11 for what happens when this is missed.

---

## Part 2: Install Caddy (reverse proxy + TLS)

### 2.1 — Install Caddy

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | tee /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install caddy
```

### 2.2 — Configure Caddyfile

```bash
cat > /etc/caddy/Caddyfile << 'EOF'
your-domain.example.com {
    handle /api/status {
        rewrite * /
        reverse_proxy localhost:18789
    }
    reverse_proxy localhost:18789
}
EOF

systemctl restart caddy
systemctl status caddy
```

The `/api/status` rewrite shim maps the DM health check path to `/` (the OpenClaw Control UI page), which returns 200 when the gateway is up. Vanilla OpenClaw has no `/api/status` endpoint — that was a moltworker-only route. This shim keeps compatibility with `ClawInstance.check_health()`.

After restart, Caddy will automatically obtain a Let's Encrypt TLS certificate. Check the logs if it doesn't work:

```bash
journalctl -u caddy --no-pager -n 50
```

### 2.3 — Control UI origin allowlist

OpenClaw's Control UI only allows connections from the gateway's own host (localhost) by default. If the Control UI shows *"origin not allowed (open the Control UI from the gateway host or allow it in gateway.controlUi.allowedOrigins)"*, the config needs updating.

The full config in step 1.4 already includes `controlUi.allowedOrigins` with your domain. If you wrote a minimal config instead, or need to add origins after the fact:

```bash
openclaw config set gateway.controlUi.allowedOrigins '["https://your-domain.example.com"]'
systemctl --user restart openclaw-gateway
```

---

## Part 3: Firewall

### Hetzner Cloud Firewall

Set these rules in the Hetzner Cloud Console (Networking → Firewalls):

| Direction | Protocol | Port | Source | Purpose |
|-----------|----------|------|--------|---------|
| Inbound | TCP | 22 | Management IPs | SSH |
| Inbound | TCP | 80 | `0.0.0.0/0` | ACME challenge (Let's Encrypt) |
| Inbound | TCP | 443 | `0.0.0.0/0` or allowlist | HTTPS (Caddy → OpenClaw) |

**If restricting port 443 to specific IPs**, you must include:
- The **DM production server's outbound IP** — find it with `curl -s ifconfig.me` from the DM server
- **Your own IP** — for Control UI browser access
- Any **VPN egress IPs** used by your team

If the Hetzner Cloud Firewall restricts port 443 to specific IPs and a user's IP isn't in the allowlist, the browser will show `ERR_CONNECTION_TIMED_OUT`. The local dev container also won't reach the server unless connected to a VPN with an allowlisted IP.

### Host firewall (UFW)

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
```

Both the Hetzner Cloud Firewall (network level, outside the server) and UFW (host level) must allow traffic for it to reach Caddy.

---

## Part 4: Validate

### 4.1 — Health check

```bash
curl -s -o /dev/null -w "%{http_code}" https://your-domain.example.com/api/status
# Expected: 200
```

### 4.2 — Control UI

Open in browser: `https://your-domain.example.com/?token=<gateway-token>`

The first browser access through Caddy will show "pairing required". Browser devices connecting through the reverse proxy are not auto-approved — only loopback connections are. Approve the browser device:

```bash
# On the server:
openclaw devices list
openclaw devices approve <requestId>
```

Each browser profile generates a unique device ID. This is a one-time step per browser. Do **not** use `dangerouslyDisableDeviceAuth` — the docs call it a "severe security downgrade" and it only affects the Control UI, not programmatic WebSocket connections.

### 4.3 — Chat test

In the Control UI, send a test message. You should get a response from the configured LLM.

**Full stack confirmed**: Browser → Caddy (TLS/Let's Encrypt) → OpenClaw Gateway → Anthropic API.

---

## Part 5: Connect to ibl-dm-pro

### 5.1 — Create claw instance record

In Django admin → Claw instances → Add:

| Field | Value |
|-------|-------|
| **Platform** | Your platform |
| **Name** | `<domain>-hetzner-<purpose>` (e.g. `prod-hetzner-primary`) |
| **Server URL** | `https://your-domain.example.com` |
| **Gateway token** | The token from step 1.3 |
| **Auth headers** | `{}` |
| **Status** | `active` |

**Naming convention**: `<domain>-<provider>-<purpose>` (e.g. `prod-hetzner-primary`, `staging-hetzner-demo`).

Run the "Health check" admin action to verify connectivity. Should return `healthy`.

### 5.2 — Generate and store device keypair

The DM backend needs an Ed25519 keypair for device identity signing. Without it, config push will fail with "missing scope: operator.read/write/admin".

Generate one:

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
from cryptography.hazmat.primitives.serialization import Encoding, NoEncryption, PrivateFormat

key = Ed25519PrivateKey.generate()
pem = key.private_bytes(Encoding.PEM, PrivateFormat.PKCS8, NoEncryption()).decode()
print(pem)
```

**Option A — Per-server (recommended for single servers)**:

Edit the claw instance record in Django admin. In the `connection_params` JSON field, add:

```json
{
  "device_identity": {
    "private_key_pem": "-----BEGIN PRIVATE KEY-----\n<base64>\n-----END PRIVATE KEY-----\n"
  }
}
```

**Option B — Global (recommended for multi-server setups)**:

Set `OPENCLAW_DEVICE_PRIVATE_KEY_PEM` in the DM server's environment or Django settings. This is the fallback when the server record has no key in metadata.

**How it works**: `OpenClawClient.connect()` receives a `connect.challenge` from the gateway, signs it with the Ed25519 key using the `v2` payload format (`v2|deviceId|clientId|clientMode|role|scopes|signedAtMs|token|nonce`), and includes the `device` object in the connect params. Fresh keypairs are auto-approved on loopback — no manual `openclaw devices approve` step needed for backend connections. Each connect signs fresh (no session token caching).

This protocol was reverse-engineered from compiled OpenClaw gateway source (`client-Bri_7bSd.js:693`, `gateway-cli-CHQJpgpN.js:19700-19760`) — no official docs exist for the signing format. The `v2` prefix is required; all signatures fail without it (discovered by trial and error).

### 5.3 — Push config

Select the server in Django admin → Admin action: **"Push config"**.

Or via shell:

```bash
docker exec web bash -c "cd /code/dl_manager && python manage.py shell -c \"
from ibl_ai_mentor.tasks import push_claw_config
push_claw_config.delay()
\""
```

Verify in Sentry/logs that the task completed without "missing scope" errors. A successful push will set `agents.files.set` (IDENTITY.md, SOUL.md), `config.get`, and `config.patch` — the gateway restarts itself after config.patch.

### 5.4 — Test chat through the SPA

1. Open the agent SPA in a browser
2. Select the Claw-backed agent
3. Send a test message (e.g. "Hello, say hi in 5 words")
4. Verify:
   - "Connected." acknowledgment appears
   - Response streams in token-by-token
   - Response completes (EOS received)
   - Message persists on page refresh (chat history saved via `_save_chat_history()`)

The full production path is: WebSocket connect → Django Channels consumer → `validate_session` → `authenticate` → `llm_runner_factory()` → `ClawLLMRunner` → `build_client_kwargs()` → `OpenClawClient.connect()` (device signing) → `chat_stream()` → `_save_chat_history()` → disconnect.

---

## Multi-Agent Setup (Optional)

The default config creates a single agent (`main`). To run multiple agents on the same gateway (e.g. tutor, course-creator, admissions), add them via the CLI:

```bash
openclaw agents add tutor-agent
openclaw agents add course-creator-agent
```

Each agent gets its own workspace (`~/.openclaw/workspace-<name>`) and agent directory (`~/.openclaw/agents/<name>/agent`). The agents appear in `agents.list` in `openclaw.json`. You can also add them by editing the config directly.

**Note**: More agents means more concurrent LLM API calls, which increases the chance of hitting provider rate limits or outages. Consider adding model fallbacks (see Step 1.4) if running multiple agents.

---

## Keeping OpenClaw Updated

Check for updates periodically:

```bash
openclaw --version          # current version
openclaw update             # update to latest
```

The gateway logs a notice on startup when an update is available. After updating, restart the service:

```bash
systemctl --user restart openclaw-gateway
```

**Caution**: OpenClaw updates may wipe the paired devices list, requiring re-pairing. See the "Device Re-Pairing" section below. Back up `~/.openclaw/` before major version upgrades.

---

## Monitoring and Diagnostics

### Live log tailing

Run in separate SSH sessions to watch both during operation:

```bash
# Gateway logs (WebSocket connects, chat requests, Anthropic API errors)
journalctl --user -u openclaw-gateway -f

# Caddy logs (incoming HTTPS requests, TLS issues)
journalctl -u caddy -f
```

### What to look for in gateway logs

| Log pattern | Meaning |
|-------------|---------|
| `protocol 3` | WebSocket handshake succeeded |
| `chat.send` | Chat request sent to LLM provider |
| `error` / `ECONNREFUSED` | Anthropic API call failed (key issue, rate limit, outage) |
| `close 4008` | WebSocket proxy issue (should not happen on Hetzner — was a moltworker/Cloudflare bug) |
| `missing scope` | Device identity signing not working — check keypair config |

### Quick health checks (no restart needed)

```bash
# Gateway alive?
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
# Expected: 200

# Structured health status
openclaw health --json

# Connected devices
openclaw devices list

# Caddy + TLS working?
curl -s -o /dev/null -w "%{http_code}" https://your-domain.example.com/api/status
# Expected: 200

# Anthropic key still valid?
curl -s -o /dev/null -w "%{http_code}" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  https://api.anthropic.com/v1/models
# Expected: 200

# Disk/memory
df -h / && free -h
```

### Verbose logging (not recommended during live use)

```bash
openclaw config set OPENCLAW_LOG_LEVEL debug
systemctl --user restart openclaw-gateway
# Remember to set back to info after debugging:
# openclaw config set OPENCLAW_LOG_LEVEL info
```

---

## Device Re-Pairing After Gateway Restarts / Updates

### The problem

OpenClaw gateway updates (`npm install -g openclaw@latest`) or gateway restarts can wipe the paired devices list. When this happens, the DM backend's device identity is no longer recognized by the gateway, and **all agents** on that server fail with `PAIRING_REQUIRED` / `NOT_PAIRED` errors.

Users see: *"The agent is starting up, please wait..."* → *"The agent is currently unavailable. Please try again later."*

This affects **all agents** linked to the server — the device identity is per claw instance, not per agent. One re-pairing fixes all agents on that server.

### How to re-pair manually

1. **Trigger a connection attempt** — send any message to any agent linked to the affected server. This creates a pending pairing request on the gateway.

2. **SSH into the OpenClaw server** and approve:

```bash
# List devices — look for the "Pending" section
openclaw devices list

# Approve the pending request (use the requestId, NOT the device ID)
openclaw devices approve <requestId>
```

3. **Retry the chat** — the next message should connect successfully. All agents on this server are now fixed.

### Why loopback auto-approval doesn't work

The design intent was that Caddy (on the same host) proxies to OpenClaw at `localhost:18789`, so connections arrive from `127.0.0.1` and are auto-approved. However, Caddy adds `X-Forwarded-For` headers with the remote client's IP, and OpenClaw uses those to determine the "real" client IP. Since the DM backend connects from a remote server, OpenClaw sees a non-loopback IP and requires manual approval.

### Solution options (for review and automation phase)

The goal is a **robust, non-fragile solution** so that gateway restarts and OpenClaw updates do not require manual re-pairing. The options below are proposed for evaluation. **No implementation is specified here** — these are directions to explore.

---

#### A. OpenClaw upstream (preferred if feasible)

**A1. Persist paired devices across restarts and version upgrades**

- **Idea**: OpenClaw stores paired devices in `~/.openclaw/devices.json` (or equivalent) and **always** loads this file on startup. On version upgrade, the gateway migrates the existing file if the schema changes rather than discarding it.
- **Pros**: Fixes the root cause for all deployments (Hetzner, Cloudflare, etc.). No platform or proxy changes.
- **Cons**: Requires OpenClaw to implement or fix persistence/migration.
- **Action**: Check OpenClaw issue tracker / docs; open a feature request or bug report if persistence is missing or broken.

**A2. Config-based trusted device registration**

- **Idea**: OpenClaw config supports a list of pre-approved device IDs or public keys (e.g. `gateway.trustedDevices: ["<deviceId>"]` or `gateway.trustedDevicePublicKeys: ["<base64url>"]`). Connections that present a valid signature for a trusted device are accepted without going through the pending-approval flow.
- **Pros**: Platform can store the device public key in config at deploy time (or push it via config push). Survives restarts and updates because it lives in config. No manual approval step.
- **Cons**: Requires OpenClaw to add this feature.

**A3. Admin API for device approval**

- **Idea**: OpenClaw gateway exposes an authenticated admin endpoint (e.g. `POST /api/admin/devices/approve`) that accepts a pending `requestId` or a device public key + scopes, and adds the device to the paired list. Protected by gateway token or a separate admin secret.
- **Pros**: Platform can automate re-pairing: when health check or connect fails with `NOT_PAIRED`, trigger a connect to create a pending request, then call the admin API to approve it.
- **Cons**: Requires OpenClaw to add and maintain this API.

---

#### B. Platform-side automation (ibl-dm-pro)

**B1. Health check detects NOT_PAIRED and alerts**

- **Idea**: The periodic OpenClaw health check (or a dedicated "deep" check) performs a full WebSocket connect attempt, not only HTTP status. If the connect fails with `NOT_PAIRED`, set server status to a distinct state (e.g. `needs_pairing`) and notify admins (email, Sentry, dashboard).
- **Pros**: Fast detection; no silent outage.
- **Cons**: Does not fix the problem by itself; only shortens time-to-detection.

**B2. Admin action: "Trigger re-pair" / "Re-pair device"**

- **Idea**: Django admin action on `ClawInstance` that (1) triggers a one-off connect attempt from the platform (to create a pending request on the gateway), and (2) instructs the admin to run `openclaw devices approve <requestId>` on the server, with the requestId shown in the admin UI or logs.
- **Pros**: Single place to start the flow; clear runbook.
- **Cons**: Still manual approval on the OpenClaw host unless combined with A3.

**B3. Fully automated re-pair via agent on OpenClaw host**

- **Idea**: A small service or cron on the OpenClaw server that (1) lists pending devices, (2) identifies the platform's device (e.g. by device ID stored in config or env), (3) calls `openclaw devices approve <requestId>`. Platform triggers this when it detects `NOT_PAIRED`.
- **Pros**: No manual step after detection.
- **Cons**: Requires secure channel from platform to OpenClaw host and maintaining the agent.

---

#### C. Infrastructure / deployment

**C1. Backup and restore `devices.json`**

- **Idea**: As part of the OpenClaw server deployment or update playbook, backup `~/.openclaw/devices.json` before an update and restore it after the new gateway starts.
- **Pros**: Works with current OpenClaw behavior. No upstream changes.
- **Cons**: Fragile if OpenClaw changes the file format; requires discipline in every update runbook.

**C2. External persistence for OpenClaw (e.g. R2 / S3)**

- **Idea**: Configure OpenClaw (if it supports it) to persist device state to an external store instead of or in addition to a local file, similar to moltworker.
- **Pros**: Survives restarts and redeploys; single source of truth.
- **Cons**: Depends on OpenClaw supporting this; more infra.

---

#### D. Reverse-proxy behavior (Caddy)

**D1. Strip forwarded headers so OpenClaw sees loopback**

- **Idea**: In Caddy, strip `X-Forwarded-For` and `X-Real-Ip` on the reverse proxy to OpenClaw so the gateway sees the connection as coming from `127.0.0.1` and applies loopback auto-approval.
- **Pros**: No OpenClaw or platform code changes. Self-healing after any restart/update.
- **Cons**: OpenClaw no longer sees the real client IP (only loopback). Relies on OpenClaw's loopback logic remaining stable and secure.

---

### Recommendation and next steps

- **Preferred direction**: Pursue **A1** and/or **A2** with the OpenClaw project so that device pairing survives restarts and updates by design. **A3** is a strong complement if the platform is to automate re-pairing without an agent on the host.
- **Short term**: Until an upstream or config-based solution exists, **manual re-pair** (see above) plus **B1** (detect and alert on `NOT_PAIRED`) reduces silent outage and makes re-pairing a clear operational step.

### Device identity scope

- Device identity is per **ClawInstance** (stored in `server.connection_params.device_identity.private_key_pem`)
- All agents linked to the same server share the same device
- One re-pairing approval covers all agents on that server
- Each server with a different keypair needs its own pairing

---

## Snags Reference

These are the issues encountered during initial deployments, collected here for quick reference. Each is described in context in the steps above.

| # | Issue | Root cause | Fix |
|---|-------|-----------|-----|
| 1 | Let's Encrypt ACME challenges fail | DNS pointed to elastic IP not routing to server; port 443 not open | Point DNS to actual server IP; open ports 80+443 before Caddy starts |
| 2 | Let's Encrypt rate limit (1 hour) | 5 failed ACME attempts from the above | Wait for cooldown; don't toggle firewall while Caddy retries |
| 3 | Control UI "origin not allowed" | OpenClaw only allows localhost origins by default | `openclaw config set gateway.controlUi.allowedOrigins '["https://..."]'` |
| 4 | Control UI "pairing required" | Browser device not auto-approved through reverse proxy | `openclaw devices approve <requestId>` (one-time per browser) |
| 5 | Browser `ERR_CONNECTION_TIMED_OUT` | Hetzner Cloud Firewall restricting port 443; user IP not in allowlist | Add IP to Hetzner firewall allowlist |
| 6 | `OPENCLAW_GATEWAY_TOKEN` not found in new SSH sessions | Token only exported in original shell | Add `export OPENCLAW_GATEWAY_TOKEN=...` to `~/.bashrc` |
| 7 | Config push "missing scope: operator.read" | `OpenClawClient` was omitting device identity from connect handshake | Implement Ed25519 device signing |
| 8 | Dev container can't reach Hetzner server | Hetzner Cloud Firewall restricts port 443; dev IP not allowlisted | Connect via VPN with allowlisted IP, or broaden firewall rule |
| 9 | Model ID mismatch | OpenClaw normalizes `claude-sonnet-4-20250514` → `claude-sonnet-4-6` | Use short alias in Django agent_config |
| 10 | DM backend `NOT_PAIRED` after gateway update | OpenClaw update/restart wiped paired devices; Caddy forwards `X-Forwarded-For` so auto-approval doesn't work | Manual re-pair (see "Device Re-Pairing" section); long-term: see "Solution options" in same section |
| 11 | Gateway dies when SSH session ends — agent replies *"The agent is starting up, please wait..."* then *"The agent is currently unavailable."* Gateway appears healthy while SSH'd in, silently dies after disconnect. | `loginctl enable-linger root` was skipped during manual setup. Without lingering, systemd kills user services when the last SSH session closes. | Run `loginctl enable-linger root` (Step 1.6). The onboard wizard handles this automatically; manual setups must do it explicitly. Verify with `loginctl show-user root 2>/dev/null | grep Linger` — should show `Linger=yes`. |