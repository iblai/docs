# Ontology & Agent Sandbox — Network Setup

How to stand up the two servers that live **inside your network** — the ontology knowledge layer and one or more agent sandboxes — when ibl.ai hosts the rest of the platform and your users reach it at `os.ibl.ai`.

This page is written for **infrastructure and network engineers**. It covers the topology, the trust boundaries, and the firewall rules. It deliberately does **not** restate the product installs — those live in the repos and are linked at each step:

| What | Repo | Guide |
|---|---|---|
| Ontology (knowledge layer) | [github.com/iblai/ontology](https://github.com/iblai/ontology) | [docs/deployment.md](https://github.com/iblai/ontology/blob/main/docs/deployment.md) · [docs/identity.md](https://github.com/iblai/ontology/blob/main/docs/identity.md) |
| Agent sandbox | [github.com/iblai/claw-setup](https://github.com/iblai/claw-setup) | [docs/server-setup.md](https://github.com/iblai/claw-setup/blob/main/docs/server-setup.md) · [docs/platform-integration.md](https://github.com/iblai/claw-setup/blob/main/docs/platform-integration.md) |

Read this page first, then run those guides with the firewall already in place.

---

## What you run, and what ibl.ai runs

| Component | Runs where | Operated by | Reachable from the internet |
|---|---|---|---|
| Platform UI + APIs (`os.ibl.ai`, `api.iblai.app`) | ibl.ai | ibl.ai | Yes — your users sign in here |
| Platform egress address (`ext.iblai.app`) | ibl.ai | ibl.ai | Yes — the single address every platform backend node calls out from |
| **Agent sandbox** — the gateway your agents execute on | **Your network** | **You** | **No** — inbound restricted to `ext.iblai.app` |
| **Ontology** — knowledge layer over your systems | **Your network** | **You** | **No** — inbound restricted to your sandbox(es) |
| Source systems — SIS/ERP, warehouse, SaaS | Already yours | You | Unchanged |

Nothing about this deployment publishes a new door into your network for the general internet. The two servers you add are reachable from exactly one place each, and both are enforced **at the firewall**, not in application config.

---

## Topology

<div style="border:1px solid #e5e7eb; border-radius:16px; padding:1.5rem 1.25rem; margin:1.75rem 0; background:#fbfbfa;">

  <div style="text-align:center; font-size:0.68rem; font-weight:700; letter-spacing:2px; text-transform:uppercase; color:#9ca3af; margin-bottom:0.35rem;">The Internet</div>
  <div style="text-align:center; font-size:0.75rem; color:#6b7280;">users sign in &middot; HTTPS 443</div>
  <div style="text-align:center; color:#d1d5db; font-size:1.1rem; line-height:1.4;">&#9660;</div>

  <div style="background:#fff; border:1px solid #e5e7eb; border-left:4px solid #2175C5; border-radius:12px; padding:1rem 1.1rem;">
    <div style="font-size:0.66rem; font-weight:700; letter-spacing:1.6px; text-transform:uppercase; color:#2175C5; margin-bottom:0.6rem;">ibl.ai &mdash; hosted platform</div>
    <div style="display:flex; flex-wrap:wrap; gap:0.5rem;">
      <span style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:0.4rem 0.7rem; font-family:ui-monospace,SFMono-Regular,Menlo,monospace; font-size:0.75rem; color:#334155;">os.ibl.ai</span>
      <span style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:0.4rem 0.7rem; font-family:ui-monospace,SFMono-Regular,Menlo,monospace; font-size:0.75rem; color:#334155;">api.iblai.app</span>
      <span style="background:#eff6ff; border:1px solid #bfdbfe; border-radius:8px; padding:0.4rem 0.7rem; font-family:ui-monospace,SFMono-Regular,Menlo,monospace; font-size:0.75rem; color:#1d4ed8; font-weight:600;">ext.iblai.app</span>
    </div>
    <div style="font-size:0.72rem; color:#6b7280; margin-top:0.6rem;">UI, APIs, auth and admin &middot; every backend node egresses from the single address <strong style="color:#1d4ed8;">ext.iblai.app</strong> &mdash; so you allowlist one address, not a fleet.</div>
  </div>

  <div style="text-align:center; color:#d1d5db; font-size:1.1rem; line-height:1.4;">&#9660;</div>

  <div style="border:1px dashed #fca5a5; background:#fef2f2; border-radius:10px; padding:0.7rem 0.9rem; text-align:center;">
    <div style="display:inline-block; background:#dc2626; color:#fff; border-radius:999px; padding:0.15rem 0.7rem; font-size:0.62rem; font-weight:700; letter-spacing:1.2px; text-transform:uppercase;">Trust boundary 1 &middot; your firewall</div>
    <div style="font-size:0.78rem; color:#991b1b; margin-top:0.5rem;"><strong>ALLOW</strong> tcp/443 from <code style="font-family:ui-monospace,SFMono-Regular,Menlo,monospace;">ext.iblai.app</code> &nbsp;&middot;&nbsp; <strong>DENY</strong> everything else</div>
  </div>

  <div style="text-align:center; color:#d1d5db; font-size:1.1rem; line-height:1.4;">&#9660;</div>

  <div style="background:#fff; border:1px solid #e5e7eb; border-left:4px solid #7c3aed; border-radius:12px; padding:1rem 1.1rem;">
    <div style="font-size:0.66rem; font-weight:700; letter-spacing:1.6px; text-transform:uppercase; color:#7c3aed; margin-bottom:0.6rem;">Your network &mdash; DMZ / sandbox tier</div>
    <div style="display:flex; flex-wrap:wrap; gap:0.6rem;">
      <div style="flex:1 1 150px; min-width:0; background:#faf5ff; border:1px solid #e9d5ff; border-radius:10px; padding:0.65rem 0.75rem;">
        <div style="font-size:0.78rem; font-weight:700; color:#6b21a8;">Sandbox 1</div>
        <div style="font-family:ui-monospace,SFMono-Regular,Menlo,monospace; font-size:0.7rem; color:#7e22ce; margin-top:0.3rem; line-height:1.6;">Caddy :443<br>gateway 127.0.0.1:18789</div>
      </div>
      <div style="flex:1 1 150px; min-width:0; background:#faf5ff; border:1px solid #e9d5ff; border-radius:10px; padding:0.65rem 0.75rem;">
        <div style="font-size:0.78rem; font-weight:700; color:#6b21a8;">Sandbox 2</div>
        <div style="font-family:ui-monospace,SFMono-Regular,Menlo,monospace; font-size:0.7rem; color:#7e22ce; margin-top:0.3rem; line-height:1.6;">Caddy :443<br>gateway 127.0.0.1:18789</div>
      </div>
      <div style="flex:1 1 150px; min-width:0; background:#faf5ff; border:1px dashed #d8b4fe; border-radius:10px; padding:0.65rem 0.75rem;">
        <div style="font-size:0.78rem; font-weight:700; color:#6b21a8;">Sandbox N</div>
        <div style="font-size:0.7rem; color:#7e22ce; margin-top:0.3rem; line-height:1.6;">add as many as you need &mdash; one per workload, tier or department</div>
      </div>
    </div>
    <div style="font-size:0.72rem; color:#6b7280; margin-top:0.6rem;">Each sandbox has its own gateway token and device key. Sandbox&#8209;to&#8209;sandbox traffic stays denied.</div>
  </div>

  <div style="text-align:center; color:#d1d5db; font-size:1.1rem; line-height:1.4;">&#9660;</div>

  <div style="border:1px dashed #fca5a5; background:#fef2f2; border-radius:10px; padding:0.7rem 0.9rem; text-align:center;">
    <div style="display:inline-block; background:#dc2626; color:#fff; border-radius:999px; padding:0.15rem 0.7rem; font-size:0.62rem; font-weight:700; letter-spacing:1.2px; text-transform:uppercase;">Trust boundary 2 &middot; your firewall</div>
    <div style="font-size:0.78rem; color:#991b1b; margin-top:0.5rem;"><strong>ALLOW</strong> tcp/443 from <strong>the sandbox addresses only</strong> &nbsp;&middot;&nbsp; <strong>DENY</strong> everything else, including ibl.ai</div>
  </div>

  <div style="text-align:center; color:#d1d5db; font-size:1.1rem; line-height:1.4;">&#9660;</div>

  <div style="background:#fff; border:1px solid #e5e7eb; border-left:4px solid #0d9488; border-radius:12px; padding:1rem 1.1rem;">
    <div style="font-size:0.66rem; font-weight:700; letter-spacing:1.6px; text-transform:uppercase; color:#0d9488; margin-bottom:0.6rem;">Your network &mdash; internal / data tier</div>
    <div style="font-size:0.85rem; font-weight:700; color:#0f766e;">Ontology host</div>
    <div style="font-family:ui-monospace,SFMono-Regular,Menlo,monospace; font-size:0.72rem; color:#0f766e; margin:0.2rem 0 0.7rem;">ontology.&lt;your-org&gt;.edu &nbsp;&middot;&nbsp; Caddy :443 &rarr; ontology-gateway :8080</div>
    <div style="display:flex; flex-wrap:wrap; gap:0.4rem;">
      <span style="background:#f0fdfa; border:1px solid #99f6e4; border-radius:8px; padding:0.35rem 0.6rem; font-size:0.72rem; color:#0f766e;">ontology-db</span>
      <span style="background:#f0fdfa; border:1px solid #99f6e4; border-radius:8px; padding:0.35rem 0.6rem; font-size:0.72rem; color:#0f766e;">vector-store</span>
      <span style="background:#f0fdfa; border:1px solid #99f6e4; border-radius:8px; padding:0.35rem 0.6rem; font-size:0.72rem; color:#0f766e;">sync-engine</span>
      <span style="background:#f0fdfa; border:1px solid #99f6e4; border-radius:8px; padding:0.35rem 0.6rem; font-size:0.72rem; color:#0f766e;">mcp-toolbox</span>
      <span style="background:#f0fdfa; border:1px solid #99f6e4; border-radius:8px; padding:0.35rem 0.6rem; font-size:0.72rem; color:#0f766e;">mcp-&lt;saas&gt;</span>
    </div>
    <div style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:0.55rem 0.75rem; margin-top:0.7rem; font-size:0.72rem; color:#475569;">
      Docker network <code style="font-family:ui-monospace,SFMono-Regular,Menlo,monospace;">ontology-internal</code> is <strong>internal:&nbsp;true</strong> &mdash; these containers have <strong>no internet route at all</strong>. Only the gateway and proxy bridge out.
    </div>
  </div>

  <div style="text-align:center; color:#d1d5db; font-size:1.1rem; line-height:1.4;">&#9660;</div>

  <div style="border:1px dashed #fcd34d; background:#fffbeb; border-radius:10px; padding:0.7rem 0.9rem; text-align:center;">
    <div style="display:inline-block; background:#d97706; color:#fff; border-radius:999px; padding:0.15rem 0.7rem; font-size:0.62rem; font-weight:700; letter-spacing:1.2px; text-transform:uppercase;">Trust boundary 3 &middot; egress allowlist</div>
    <div style="font-size:0.78rem; color:#92400e; margin-top:0.5rem;"><strong>ALLOW</strong> only the named source systems and ports &nbsp;&middot;&nbsp; <strong>no general internet egress</strong></div>
  </div>

  <div style="text-align:center; color:#d1d5db; font-size:1.1rem; line-height:1.4;">&#9660;</div>

  <div style="display:flex; flex-wrap:wrap; gap:0.5rem; justify-content:center;">
    <span style="background:#fff; border:1px solid #e5e7eb; border-top:3px solid #2175C5; border-radius:8px; padding:0.45rem 0.7rem; font-size:0.73rem; color:#334155;">PeopleSoft / Oracle <span style="color:#94a3b8;">1521</span></span>
    <span style="background:#fff; border:1px solid #e5e7eb; border-top:3px solid #2175C5; border-radius:8px; padding:0.45rem 0.7rem; font-size:0.73rem; color:#334155;">Snowflake <span style="color:#94a3b8;">443</span></span>
    <span style="background:#fff; border:1px solid #e5e7eb; border-top:3px solid #2175C5; border-radius:8px; padding:0.45rem 0.7rem; font-size:0.73rem; color:#334155;">Postgres <span style="color:#94a3b8;">5432</span></span>
    <span style="background:#fff; border:1px solid #e5e7eb; border-top:3px solid #7c3aed; border-radius:8px; padding:0.45rem 0.7rem; font-size:0.73rem; color:#334155;">Canvas <span style="color:#94a3b8;">443</span></span>
    <span style="background:#fff; border:1px solid #e5e7eb; border-top:3px solid #7c3aed; border-radius:8px; padding:0.45rem 0.7rem; font-size:0.73rem; color:#334155;">Salesforce <span style="color:#94a3b8;">443</span></span>
    <span style="background:#fff; border:1px solid #e5e7eb; border-top:3px solid #7c3aed; border-radius:8px; padding:0.45rem 0.7rem; font-size:0.73rem; color:#334155;">ServiceNow <span style="color:#94a3b8;">443</span></span>
  </div>

</div>

**The rule that makes this design work:** each hop is allowed to talk to exactly one thing downstream, and nothing upstream can skip a hop. The platform cannot reach the ontology. The ontology cannot reach the internet. A compromised agent cannot reach a database directly — it can only call MCP tools the ontology chooses to expose, read-only, scoped to the calling user's role.

---

## Boundary 1 — the platform to your sandbox

The platform initiates the connection *into* your sandbox (WebSocket, Ed25519 device-identity signed). Your sandbox never dials out to the platform, so this is a pure inbound rule.

### `ext.iblai.app` is an egress address, not a server

This is the one thing to get right before writing the rule. **`ext.iblai.app` is not where your agents are hosted, and it is not a single machine.** It is the address that *every* backend node in the ibl.ai platform makes its outbound connections from.

It exists so that you have one thing to allowlist. The alternative is enumerating the IP of each backend node — there are many of them, and the set changes as nodes are added, replaced or scaled. A per-node list would need constant maintenance and would break your agents the moment it drifted. One stable egress address removes that whole class of failure.

Resolve it and pin what it returns:

```bash
dig +short ext.iblai.app
```

> **Confirm before you pin.** Resolve it yourself at deployment time and have ibl.ai confirm the current egress address for your tenant. If your firewall supports FQDN objects, use `ext.iblai.app` rather than a literal IP, so a future change doesn't take your agents down.

### Sandbox host — inbound rules

| Port | Source | Purpose | Notes |
|---|---|---|---|
| 443/tcp | `ext.iblai.app` only | Platform → gateway (WSS + HTTPS) | The only production door — one address covering every backend node |
| 80/tcp | *closed* — see TLS note | ACME `http-01` challenge | Only if you cannot use DNS-01 |
| 22/tcp | Your bastion / admin CIDR | Administration | Never `0.0.0.0/0` |
| — | Everything else | — | **DENY** |

```bash
# ufw — default deny, then two explicit allows
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 203.0.113.10/32 to any port 22 proto tcp comment 'admin bastion'
sudo ufw allow from <ext.iblai.app IP>/32 to any port 443 proto tcp comment 'ibl.ai platform egress'
sudo ufw enable
```

### Sandbox host — outbound rules

| Destination | Port | Purpose |
|---|---|---|
| Ontology host | 443/tcp | MCP queries against your data |
| LLM provider endpoint | 443/tcp | Model inference (skip entirely if you self-host the model) |
| Your DNS + NTP | 53, 123 | Resolution and clock (clock skew breaks JWT validation) |
| Package/registry mirrors | 443/tcp | Install and updates — can be closed after provisioning |

### TLS without opening the server to the world

The sandbox setup guide assumes ports 80 and 443 are open to `0.0.0.0/0` so Let's Encrypt can validate. **That conflicts with this topology**, and it is the single most common way this deployment ends up more exposed than intended.

Pick one, in order of preference:

1. **DNS-01 challenge.** Caddy proves domain control through a DNS TXT record, so no inbound port has to be open to the internet at all. Requires a Caddy build that includes your DNS provider module (`xcaddy build --with github.com/caddy-dns/<provider>`) and an API credential scoped to that one zone.
2. **Your own internal CA / institution-issued certificate.** Point Caddy at the cert and key with `tls <cert> <key>` and disable automatic HTTPS. ibl.ai must trust the issuing chain — raise this with us before you deploy.
3. **`http-01` with port 80 open, 443 restricted.** Acceptable but weaker: Let's Encrypt validates from many IPs, so port 80 must accept the world. Only the ACME endpoint is served there. If you take this path, keep port 80 open *only* during issuance and renewal windows.

> Do **not** toggle firewall rules while Caddy is retrying an ACME challenge — after five failures Let's Encrypt rate-limits the hostname for an hour.

### Multiple sandboxes

Run as many as you need — one per workload, per department, per data-sensitivity tier, or simply for capacity. Each is an independent host with the same two rules (inbound 443 from `ext.iblai.app`, outbound 443 to the ontology) and its own gateway token and device key. They do **not** need to reach each other; leave sandbox-to-sandbox traffic denied.

Register each one with the platform separately per [platform-integration.md](https://github.com/iblai/claw-setup/blob/main/docs/platform-integration.md). Add every sandbox's address to the ontology's inbound allowlist as you add it — that list is the one place this topology needs maintaining.

---

## Boundary 2 — your sandbox to the ontology

This is the boundary the whole design exists to protect. The ontology host holds read credentials for your systems of record, so it gets the tightest rules on the network.

### Ontology host — inbound rules

| Port | Source | Purpose |
|---|---|---|
| 443/tcp | **Sandbox IPs only** | MCP over HTTPS → Caddy → `ontology-gateway:8080` |
| 443/tcp | Admin CIDR (optional) | Admin dashboard on its own hostname |
| 22/tcp | Your bastion / admin CIDR | Administration |
| — | Everything else, **including the internet and the rest of your corporate LAN** | **DENY** |

```bash
sudo ufw default deny incoming
sudo ufw allow from 203.0.113.10/32 to any port 22 proto tcp comment 'admin bastion'
sudo ufw allow from 10.20.0.11/32 to any port 443 proto tcp comment 'sandbox 1'
sudo ufw allow from 10.20.0.12/32 to any port 443 proto tcp comment 'sandbox 2'
sudo ufw enable
```

> ### Docker publishes ports *around* ufw — this will bite you
>
> The ontology stack runs `proxy` with `ports: ["443:443"]`. Docker writes its own
> `iptables` rules into the `DOCKER-USER` chain, which is evaluated **before** ufw's
> rules. A `ufw deny` on 443 therefore does **not** block a published container port,
> and the ontology will be reachable from anywhere that can route to the host even
> though `ufw status` looks correct.
>
> Close it properly, both ways:
>
> **1. Bind the published port to one interface** — in your `docker-compose.override.yml`:
> ```yaml
> services:
>   proxy:
>     ports: ["10.30.0.5:443:443"]   # private NIC only, never 0.0.0.0
> ```
>
> **2. Filter in `DOCKER-USER`**, which Docker will not overwrite:
> ```bash
> # allow the sandboxes, drop everything else headed for the container network
> sudo iptables -I DOCKER-USER -s 10.20.0.11/32 -p tcp --dport 443 -j RETURN
> sudo iptables -I DOCKER-USER -s 10.20.0.12/32 -p tcp --dport 443 -j RETURN
> sudo iptables -A DOCKER-USER -p tcp --dport 443 -j DROP
> ```
> Persist the rules (`iptables-persistent`, or your config-management equivalent) —
> an unpersisted `DOCKER-USER` chain is empty again after reboot, which is a silent
> re-opening of the boundary.
>
> Verify from a host that is *not* a sandbox — see [Prove it is actually closed](#prove-it-is-actually-closed).

### The stack's own internal segmentation

The compose file ships with two Docker networks, and you should not flatten them:

- **`ontology-internal`** is declared `internal: true` — the databases, the MCP servers, and the sync engine on it **cannot reach the internet at all**, in either direction.
- **`ontology-exposed`** is a normal bridge. Only `ontology-gateway` and `proxy` sit on both.

So your source credentials, the Postgres cache, and the materialized memory files are never on a network with an internet route. The gateway also mounts the memory volume read-only (`:ro`). Full detail in [docs/deployment.md § internal vs. exposed networks](https://github.com/iblai/ontology/blob/main/docs/deployment.md).

One consequence worth planning for: **a cloud source system is unreachable from `ontology-internal`.** If you connect Snowflake, Canvas, Salesforce or any other SaaS, that MCP container needs an egress-capable network — which is what Boundary 3 is for.

---

## Boundary 3 — the ontology to your data sources

The ontology host should have **no general internet egress**. Replace it with an allowlist of the specific systems it reads.

| Destination | Port | Typical source |
|---|---|---|
| `psft-db.internal.<org>` | 1521/tcp | PeopleSoft / Oracle |
| `<account>.snowflakecomputing.com` | 443/tcp | Snowflake |
| `<host>` | 5432 / 3306 / 1433 | Postgres / MySQL / SQL Server |
| `<org>.instructure.com` | 443/tcp | Canvas LMS |
| `<org>.my.salesforce.com`, `<org>.service-now.com` | 443/tcp | Salesforce, ServiceNow |
| `login.microsoftonline.com` | 443/tcp | Entra ID — JWKS fetch for token validation |
| Your DNS + NTP | 53, 123 | Resolution and clock |

Two rules that make this list safe to hand to a security reviewer:

- **Every database credential is read-only, and the tooling proves it.** Before a source is provisioned, `ontology service test <name>` attempts CREATE, INSERT, UPDATE, DELETE, DROP, ALTER and TRUNCATE. All seven must be denied or provisioning refuses to continue. See [docs/read-only-db-user.md](https://github.com/iblai/ontology/blob/main/docs/read-only-db-user.md) for the grants to issue.
- **Entra ID is the only identity in play.** Every MCP request carries the end user's token; the gateway validates it and resolves the caller's role from `roles.yaml`, so an agent sees only what that human is entitled to. Detail in [docs/identity.md](https://github.com/iblai/ontology/blob/main/docs/identity.md).

Keep `login.microsoftonline.com` reachable — without JWKS the gateway cannot validate a single token and every request fails closed.

---

## Build order

Do it in this sequence. Each step is verifiable before the next one can hurt you.

1. **Provision both hosts** with private addresses and DNS records. Ontology: Ubuntu 22.04+, 16 GB RAM, 500 GB disk. Sandbox: 2 vCPU / 4 GB for the standard gateway, or 4+ vCPU / 16 GB for the containerized GPU-host variant — the sandbox repo states the requirement for the one you pick.
2. **Apply the firewall rules first, before installing anything.** Provisioning a service and then closing the door leaves a window where it was open.
3. **Create the read-only database users** and issue the Entra app registration.
4. **Deploy the ontology** — [docs/deployment.md](https://github.com/iblai/ontology/blob/main/docs/deployment.md). Run `ontology service test <name>` per source, then `ontology health` and `ontology config validate`.
5. **Deploy each sandbox** — [docs/server-setup.md](https://github.com/iblai/claw-setup/blob/main/docs/server-setup.md), with your chosen TLS path instead of the default open-to-the-world ACME.
6. **Open Boundary 2** — add each sandbox address to the ontology's inbound allowlist and confirm from the sandbox: `curl -sv https://ontology.<your-org>.edu/mcp`.
7. **Register with the platform** — the MCP server, then each sandbox instance. See the [ontology platform-integration guide](https://github.com/iblai/ontology/blob/main/docs/platform-integration.md) and the [sandbox platform-integration guide](https://github.com/iblai/claw-setup/blob/main/docs/platform-integration.md).
8. **Re-run the closure checks below** after everything is live, not just before.

### A decision you have to make at step 7

The platform can register an MCP server by **automated discovery** — it fetches the OAuth protected-resource metadata from your ontology URL (RFC 9728). That fetch originates from **ibl.ai**, not from your sandbox, so under the topology above it is blocked by Boundary 2.

Choose deliberately:

- **Keep Boundary 2 closed** (recommended) and register the ontology with explicit static configuration — issuer, endpoints and scopes supplied by hand — so no ibl.ai-originated request ever needs to reach the ontology.
- **Add `ext.iblai.app` to the ontology's inbound allowlist**, which is a legitimate posture if you also want platform backend nodes to query the ontology directly rather than only through a sandbox. It is a wider boundary; make it a decision, not an accident.

Raise this with ibl.ai before you register — it determines the payload we ask you for.

---

## Prove it is actually closed

Firewall rules that were never tested from the outside are a hope, not a control. Run all four from a host that is **not** a sandbox and **not** on the admin CIDR.

```bash
# 1 · Ontology must be unreachable from anywhere but the sandboxes
nmap -Pn -p 443 ontology.<your-org>.edu          # expect: filtered
curl -m 5 -sv https://ontology.<your-org>.edu/mcp # expect: timeout, not a TLS handshake

# 2 · Sandbox must be unreachable from anywhere but ext.iblai.app
nmap -Pn -p 443 <sandbox-fqdn>                    # expect: filtered

# 3 · From the sandbox, the ontology must answer
ssh sandbox1 'curl -m 5 -so /dev/null -w "%{http_code}\n" https://ontology.<your-org>.edu/mcp'

# 4 · From the ontology host, general egress must fail
ssh ontology 'curl -m 5 -sI https://example.com'  # expect: failure
```

A TLS handshake in check 1 or 2 means the boundary is open — most often the Docker/ufw interaction above, or a cloud security group that was never narrowed to match the host firewall. **Both layers must be right**; the host firewall alone is not enough on a cloud VM, and the security group alone is not enough on a Docker host.

Re-run these after any Docker or Compose upgrade. Docker rewrites its iptables rules on restart, and a `DOCKER-USER` chain that was configured but not persisted comes back empty.

---

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Ontology answers from an unexpected host | Docker published 443 past ufw | Bind the port to the private NIC and add `DOCKER-USER` rules; persist them |
| Every MCP request 401s | Entra JWKS unreachable, or host clock skew | Allow `login.microsoftonline.com:443`; check NTP |
| `mcp-toolbox` crash-loops on start | The Toolbox connects to **every** source at startup and refuses to start if one is unreachable | Keep only configured, reachable sources in the active `tools.yaml` |
| A SaaS source times out but a database works | That MCP container is on `ontology-internal` (`internal: true`, no egress) | Put it on an egress-capable network in your override file |
| ACME fails repeatedly, then rate-limits | `http-01` cannot work behind a closed port 80 | Move to DNS-01 or an internally issued certificate |
| Agent chat connects but has no permissions | Ed25519 device identity missing from the handshake — the gateway grants zero scopes | See the device-identity section of the sandbox server setup guide |
| Sandbox works, then stops after a platform change | The pinned `ext.iblai.app` egress address changed | Use an FQDN firewall object, and confirm the current egress address with ibl.ai |

---

## What to have ready before you call ibl.ai

- The FQDN of the ontology (`ontology.<your-org>.edu`) and of each sandbox.
- The **outbound** addresses of your sandboxes, so we can confirm what will reach the ontology.
- Your Entra tenant ID and the ontology app registration's client ID.
- Which TLS path you chose, and — if it is a private CA — the issuing chain.
- Your answer to the discovery decision at step 7.
- Your org key and platform admin credentials for registration.

## Related

- [Platform Architecture — Infrastructure & Data Flow](https://ibl.ai/architecture#infrastructure) — where these two servers sit in the full deployed stack
- [Infrastructure Architecture](/developer/infrastructure/infra-architecture) — the platform side, if you are also self-hosting it
- [Sandbox server setup](/developer/claw-setup/server-setup) · [Sandbox server setup on a GPU host](/developer/claw-setup/nemoclaw-setup) · [Sandbox platform integration](/developer/claw-setup/platform-integration)
- [MCP Servers](/developer/applications/mcp) and [MCP & OAuth Architecture](/developer/agents/mcp-authentication/architecture) — how the platform models an MCP server connection
