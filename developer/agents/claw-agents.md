# Claw Agents

Pre-built agent configurations for [OpenClaw](https://github.com/iblai/iblai-claw-setup) and [NemoClaw](https://github.com/NVIDIA/NemoClaw) instances, organized by vertical.

---

## Overview

**claws** is a curated library of ready-to-deploy agent configurations, one bundle per ibl.ai solution segment. Each segment is a complete multi-agent system — a parent orchestrator plus a roster of specialist subagents — packaged so a claw host can import it with no conversion step.

The repository ships **7 segment configurations totalling 95 agents and 76 tool skills**. Every agent arrives pre-configured with its system prompts, tool selections, and behavioral parameters.

| Segment | Parent agent | Subagents | Tool skills |
|---------|--------------|-----------|-------------|
| `higher-education` | Campus Assistant | 16 | 10 |
| `k-12` | School Assistant | 12 | 12 |
| `enterprise` | Workplace Assistant | 12 | 12 |
| `government` | Agency Assistant | 12 | 10 |
| `legal` | Firm Assistant | 12 | 11 |
| `financial-services` | Advisory Assistant | 12 | 11 |
| `medical-healthcare` | Care Assistant | 12 | 10 |

---

## How a segment is structured

Each segment has one **parent agent** — the default entry point — and a set of **specialist subagents**. The parent interprets each request and delegates to the right specialist through the `sessions_spawn` tool, then synthesizes the results. The subagents a parent may spawn are declared in its `subagents.allowAgents` list.

A segment directory contains `openclaw.json` (the gateway config covering every agent, model, sandbox, and skill), a `.config-hash`, an `.env.example` credential template, a shared `workspace/`, a `skills/` directory with one `SKILL.md` per third-party tool, and an `agents/` tree. Each agent's narrative definition lives in Markdown: `IDENTITY.md`, `SOUL.md`, `TOOLS.md`, plus optional `AGENTS.md`, `BOOTSTRAP.md`, `HEARTBEAT.md`, and `MEMORY.md` files.

This is a configuration-only repository — no application code, build system, or tests.

---

## Repository

- **GitHub**: [iblai/claws](https://github.com/iblai/claws)
- **License**: MIT

---

## Getting Started

Each segment is its own repository aggregated here as a git submodule, so you must clone recursively — a plain `git clone` leaves the segment directories empty:

```bash
git clone --recurse-submodules https://github.com/iblai/claws.git
cd claws
```

If you already cloned without submodules, run `git submodule update --init` to populate them.

Browse the segment directories to find the roster suited to your use case, then copy the segment's contents into `/sandbox/.openclaw/` on your claw host, recompute the config hash, and reload the gateway.

---

## Related

- [Claw Setup](/developer/agents/claw-setup) — stand up the OpenClaw or NemoClaw server these configs deploy to
- [.iblai Agent Standard](/developer/agents/standard) — the portable agent definition format
