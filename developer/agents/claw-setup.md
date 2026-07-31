# Claw Setup

Connect self-hosted [Claw](https://github.com/iblai/iblai-claw-setup) servers — OpenClaw or NVIDIA NemoClaw — to the ibl.ai platform.

---

## Overview

**iblai-claw-setup** runs your own AI agent infrastructure while you manage it through ibl.ai's APIs and applications. The compute stays on hardware you control; the configuration, skills, and analytics are administered centrally.

Once an instance is connected it becomes reachable from every ibl.ai application and from any custom integration built on the platform REST API. You configure agent identities, push skills, assign them, and chat with users — all without granting the platform access to your servers.

---

## Features

- **Self-hosted agents** — run OpenClaw or NemoClaw on your own infrastructure
- **Automatic TLS** — a Caddy reverse proxy handles Let's Encrypt certificates with zero configuration
- **Centralized configuration** — manage agent identities, personalities, and behavioral guidelines over the API
- **Skill system** — build reusable skills with scripts and resources, then push them to instances
- **Multi-model support** — Anthropic, OpenRouter, or any OpenAI-compatible provider, with automatic fallbacks
- **Multi-agent deployments** — several agents on a single gateway
- **Secure by default** — Ed25519 device identity signing, loopback-only gateway binding, token-based auth
- **Monitoring** — health checks, connectivity tests, security audits, and version tracking via the API

---

## Repository

- **GitHub**: [iblai/iblai-claw-setup](https://github.com/iblai/iblai-claw-setup)
- **License**: MIT

---

## Getting Started

On a fresh Debian or Ubuntu server, the installer handles both the server side and the platform registration. `HARNESS_TYPE` is the only variable you set — everything else is asked interactively:

```bash
# OpenClaw (default)
curl -fsSL https://raw.githubusercontent.com/iblai/claw-setup/main/install.sh | bash

# NemoClaw
curl -fsSL https://raw.githubusercontent.com/iblai/claw-setup/main/install.sh | HARNESS_TYPE=nemoclaw bash
```

Point your domain's DNS A record at the server and open ports 80 and 443 before you run it. The script prompts for the domain, the LLM provider and API key, sandbox and firewall choices, and whether to register the instance on ibl.ai. It then installs the claw runtime, Caddy with automatic TLS, the firewall, and the extensions plugin, and prints the gateway token and Ed25519 device key.

Answers are cached and offered as defaults next time, and re-running is safe — it never regenerates an existing token or key.

---

## Related

- [Claw Agents](/developer/agents/claw-agents) — pre-built agent rosters to deploy onto your server
- [.iblai Agent Standard](/developer/agents/standard) — the portable agent definition format
