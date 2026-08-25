# Claw Agents

> Mirrored from [`iblai/claws`](https://github.com/iblai/claws) · [`README.md`](https://github.com/iblai/claws/blob/main/README.md). This page is generated — edit it in the repository, not here.

Drop-in [OpenClaw](https://github.com/iblai/iblai-claw-setup) / [NemoClaw](https://github.com/NVIDIA/NemoClaw) agent configurations for every [ibl.ai](https://ibl.ai) solution segment. Each segment is a complete multi-agent system — one parent orchestrator plus a roster of specialist subagents — packaged so a NemoClaw host can import it with zero conversion.

[OpenClaw](https://github.com/iblai/iblai-claw-setup)
[NemoClaw](https://github.com/NVIDIA/NemoClaw)
[ibl.ai Platform](https://ibl.ai)
[License: MIT](https://github.com/iblai/claws/blob/main/LICENSE)

---

## Overview

This repository contains **7 segment configurations**, one for each segment in the ibl.ai Solutions menu. Each segment is its own repository (`github.com/iblai/<segment>-agents`), aggregated here as a **git submodule** — so the `<segment>/` directories populate only after a recursive clone (`git clone --recurse-submodules`) or `git submodule update --init`. Every segment is a self-contained OpenClaw gateway config: copy its contents into `/sandbox/.openclaw/` on a NemoClaw host, recompute one hash, and reload the gateway.

| Segment | Parent agent | Subagents | Tool skills |
|---------|--------------|-----------|-------------|
| [`higher-education`](https://github.com/iblai/higher-education-agents) | Campus Assistant | 16 | 10 |
| [`k-12`](https://github.com/iblai/k-12-agents) | School Assistant | 12 | 12 |
| [`enterprise`](https://github.com/iblai/enterprise-agents) | Workplace Assistant | 12 | 12 |
| [`government`](https://github.com/iblai/government-agents) | Agency Assistant | 12 | 10 |
| [`legal`](https://github.com/iblai/legal-agents) | Firm Assistant | 12 | 11 |
| [`financial-services`](https://github.com/iblai/financial-services-agents) | Advisory Assistant | 12 | 11 |
| [`medical-healthcare`](https://github.com/iblai/medical-healthcare-agents) | Care Assistant | 12 | 10 |

**95 agents** (7 parents + 88 specialists) and **76 tool skills** in total. This is a configuration-only repository — no application code, build system, or tests.

## How a segment is structured

Each segment has one **parent agent** (the default entry point) and a set of **specialist subagents**. The parent interprets each request and delegates to the right specialist via the `sessions_spawn` tool; the subagents it may spawn are listed in its `subagents.allowAgents`. The parent then synthesizes the results.

```
<segment>/
├── README.md                  # segment overview + import steps
├── openclaw.json              # gateway config — every agent, model, sandbox, skills
├── .config-hash               # sha256 of openclaw.json
├── .env.example               # template for ~/.openclaw/.env (sample credentials)
├── workspace/
│   └── .gitkeep               # shared writable workspace (empty at install)
├── skills/
│   └── <tool>/
│       └── SKILL.md           # one skill directory per major third-party tool
└── agents/
    └── <agent-id>/
        └── agent/
            ├── IDENTITY.md          # persona — name, role, vibe
            ├── SOUL.md              # behavioral guidelines
            ├── TOOLS.md             # tool/integration notes + data sources
            ├── AGENTS.md            # multi-agent routing table (parent only)
            ├── USER.md              # user/environment context    (optional)
            ├── BOOTSTRAP.md         # one-time first-run setup     (optional)
            ├── HEARTBEAT.md         # periodic awareness checklist (optional)
            ├── MEMORY.md            # seed long-term facts         (optional)
            └── auth-profiles.json   # per-agent provider credentials (sample)
```

### Runtime-only by design

The repo contains **only what the OpenClaw/NemoClaw runtime actually loads** — the per-agent files above plus `openclaw.json`, `.config-hash`, `skills/`, and `.env.example`. Model, instance config, sandbox/security policy, and skill wiring all live in `openclaw.json`; there is no `MODEL.md`, `CONFIG.json`, `SECURITY.md`, or `DATA.md`, because the runtime never read them. Optional files (`USER.md`, `BOOTSTRAP.md`, `HEARTBEAT.md`, `MEMORY.md`) appear only on agents that genuinely need them.

### `openclaw.json`

The OpenClaw gateway configuration. Every agent — parent and subagents — is declared in `agents.list[]`. The parent carries `default: true` and a `subagents.allowAgents` list naming every child. `agents.defaults` holds the shared `model`, `subagents` limits, the `skills` array, and the `sandbox` block (the runtime's security policy). Each agent entry points at its workspace via `agentDir`, and carries its `model` and any `heartbeat`/`session` settings. After any edit to `openclaw.json`, recompute `.config-hash`.

### Skills & credentials

`skills/` holds one skill **directory** per major platform popular in the segment (Canvas, Banner, Salesforce, ServiceNow, Epic FHIR, Westlaw, and so on) — each a `<tool>/SKILL.md` with `name`, `description`, and a single-line `metadata` declaring the env vars it needs. `agents.defaults.skills` in `openclaw.json` wires them to the agents by name. Skill credentials resolve from environment variables; `.env.example` is the committed template for the OpenClaw daemon env file `~/.openclaw/.env`.

Both `auth-profiles.json` (per-agent LLM provider keys) and `.env.example` (per-tool API credentials) ship with **clearly-marked, non-functional sample placeholders**. Replace them with real values before deploying.

## Installing a segment

Each segment directory drops onto an OpenClaw/NemoClaw host with no conversion. Set the host up first with the [iblai-claw-setup](https://github.com/iblai/iblai-claw-setup) guide (`docs/server-setup.md`) — that installs the gateway, the `openclaw-gateway` service, and the config root.

The config root is **`/sandbox/.openclaw/`** on a NemoClaw sandbox host, or **`~/.openclaw/`** for a standalone OpenClaw install. Substitute whichever applies in the steps below.

```bash
# 1. Get the segment — each segment is its own repo
git clone https://github.com/iblai/higher-education-agents.git
cd higher-education-agents
#    (or grab all 7 at once: git clone --recurse-submodules https://github.com/iblai/claws.git)

# 2. Copy the config into the config root, excluding git metadata (NemoClaw path shown)
cp -r openclaw.json .config-hash .env.example agents skills workspace /sandbox/.openclaw/
#    these land directly at the root — openclaw.json, agents/, skills/, workspace/

# 3. Give the sandbox user ownership of the agent workspaces (NemoClaw only)
chown -R sandbox:sandbox /sandbox/.openclaw/agents/ /sandbox/.openclaw/workspace/

# 4. Fill in real credentials
cp /sandbox/.openclaw/.env.example ~/.openclaw/.env
vi ~/.openclaw/.env                       # third-party platform API keys
#    and replace the sample apiKey in every agents/*/agent/auth-profiles.json

# 5. Recompute the config hash (the gateway refuses to start on a mismatch)
cd /sandbox/.openclaw && shasum -a 256 openclaw.json > .config-hash

# 6. Reload the gateway
openclaw gateway reload                   # or: systemctl restart openclaw-gateway

# 7. Verify the parent agent is live
openclaw agent list                       # <segment>-assistant shows default: true, ready
```

Copying a segment installs its **parent orchestrator and every specialist subagent at once** — they are all declared in that segment's `openclaw.json`. Each segment's own `README.md` carries the full roster and step-by-step.

## Installing a segment with Claude Code

If Claude Code (or any agent that can run a shell) is running on the OpenClaw/NemoClaw host, hand it the runbook below — it points at this repo and installs a full segment, parent plus all subagents, into the live environment.

> **Install runbook — paste into Claude Code on the host**
>
> Install the `<segment>` agent configuration from its repo at `https://github.com/iblai/<segment>-agents` into this OpenClaw/NemoClaw environment. `<segment>` is one of: `higher-education`, `k-12`, `enterprise`, `government`, `legal`, `financial-services`, `medical-healthcare`.
>
> 1. Confirm the OpenClaw config root: `/sandbox/.openclaw/` on a NemoClaw sandbox host, `~/.openclaw/` for standalone OpenClaw. Confirm the gateway is installed (`openclaw --version`); if not, stop and point the user to https://github.com/iblai/iblai-claw-setup.
> 2. Clone the segment's repo to a temp dir: `git clone https://github.com/iblai/<segment>-agents.git /tmp/<segment>-agents` (or `git -C /tmp/<segment>-agents pull` if already present).
> 3. Copy the segment's config files into the config root, skipping `.git/`: `cp -r /tmp/<segment>-agents/{openclaw.json,.config-hash,.env.example,agents,skills,workspace} <config-root>/`. They must land directly in the root, not in a subdirectory.
> 4. On a NemoClaw host, set ownership: `chown -R sandbox:sandbox <config-root>/agents/ <config-root>/workspace/`.
> 5. Credentials — never invent values. Copy `<config-root>/.env.example` to `~/.openclaw/.env` and tell the user which variables need real keys. Every `agents/*/agent/auth-profiles.json` ships a sample placeholder `apiKey`; tell the user to replace each with a real Anthropic API key. Do not start the gateway until the user confirms credentials are in place.
> 6. Recompute the hash from inside the config root: `cd <config-root> && shasum -a 256 openclaw.json > .config-hash`.
> 7. Reload the gateway: `openclaw gateway reload` (or `systemctl restart openclaw-gateway`).
> 8. Verify: `openclaw agent list` — the `<segment>-assistant` parent should show `default: true` and status `ready`, alongside every subagent from that segment.
>
> Report which segment was installed, how many agents are now registered, and any credential variables still left as placeholders.

This installs the **whole segment** — one parent and all its specialists. To run more than one segment on the same host, the segments' `openclaw.json` files must be merged rather than copied over each other; that is a manual step beyond this runbook.

## Agent rosters

<details>
<summary>Higher Education — Campus Assistant + 16 specialists</summary>

Enrollment · Application Reader · Financial Aid · Academic Advisor · Tutoring · Retention · Student Services · Career Services · Faculty · Research · Alumni · IT Help Desk · Prospective Student · Yield · Continuing Education · Administrative
</details>

<details>
<summary>K-12 — School Assistant + 12 specialists</summary>

Tutoring · Lesson Planning · Assessment · Writing Feedback · Special Education · Content Creation · Student Safety · Family Communication · Curriculum Alignment · Professional Development · Research · Administration
</details>

<details>
<summary>Enterprise — Workplace Assistant + 12 specialists</summary>

Knowledge · IT Help Desk · Sales Enablement · Customer Support · Onboarding · HR · Marketing · Engineering · Meeting · Training · Data Analysis · Operations
</details>

<details>
<summary>Government — Agency Assistant + 12 specialists</summary>

Citizen Services · Knowledge · IT Help Desk · Employee Training · Compliance · Legislative · Budget · HR · Procurement · Onboarding · Constituent Communication · Security
</details>

<details>
<summary>Legal — Firm Assistant + 12 specialists</summary>

Case Research · Contract Review · Client Intake · Billing & Time · Discovery · Compliance · Knowledge · Brief Drafting · Conflicts Check · Training · Docket Management · IT Help Desk
</details>

<details>
<summary>Financial Services — Advisory Assistant + 12 specialists</summary>

Compliance · Risk Assessment · Client Advisory · Regulatory Reporting · Portfolio Analysis · Employee Training · KYC/AML · Fraud Detection · Client Onboarding · Knowledge · IT Help Desk · Operations
</details>

<details>
<summary>Medical / Healthcare — Care Assistant + 12 specialists</summary>

Clinical Support · Patient Education · Medical Coding · Compliance Training · Research · Provider Onboarding · Prior Authorization · Care Coordination · Documentation · Quality Improvement · IT Help Desk · Knowledge Management
</details>

## Adding an agent to a segment

1. Create `<segment>/agents/<agent-name>-agent/agent/` with `IDENTITY.md`, `SOUL.md`, `TOOLS.md`, and `auth-profiles.json` (add `USER.md`/`BOOTSTRAP.md`/`HEARTBEAT.md`/`MEMORY.md` only if warranted).
2. Add an entry to `agents.list[]` in `<segment>/openclaw.json` (set `id`, `name`, `agentDir`, `model`, `identity`, `tools`).
3. Add the new id to the parent agent's `subagents.allowAgents` and to its `AGENTS.md` routing table.
4. Recompute the hash: `sha256sum openclaw.json > .config-hash`.
5. Update the segment `README.md` and the roster in this file.

## Adding a new segment

Mirror an existing segment directory: a parent agent plus specialist subagents, an `openclaw.json` declaring them all, a `skills/` directory of `<tool>/SKILL.md` directories, `.env.example`, a `README.md`, and a recomputed `.config-hash`.

## License

Released under the [MIT License](https://github.com/iblai/claws/blob/main/LICENSE).
