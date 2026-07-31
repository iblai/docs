# .iblai Agent Standard

The [.iblai Agent File Format](https://github.com/iblai/standard) is a Markdown-based portable agent definition format.

---

## Overview

**standard** defines the `.iblai` file format: a Markdown-based specification that captures everything needed to define, share, and load an AI agent in a single file. Because the format is plain Markdown, an agent definition stays human-readable, reviewable in a pull request, and diffable in version control — no proprietary schema, no vendor database.

The goal is portability. The same `.iblai` file describes an agent across tools, teams, and platforms, so an agent authored once can be loaded by any runtime that implements the specification rather than rebuilt per vendor. That property is what keeps agent definitions from becoming a lock-in surface.

The specification is at **v1.0.0** and released under the MIT license.

---

## What a definition covers

An `.iblai` file is organized into sections, each describing one facet of the agent:

- **Agent info** — name, description, version, and metadata
- **Soul** — voice, temperament, values, and ethical boundaries
- **Identity** — the persona the agent presents and the role it fills
- **Tools** — the integrations and data sources it may reach for
- **Routing** — for orchestrators, which specialist agents it may delegate to

The repository ships an interactive generator that walks you through these sections and emits a valid file, so you can produce a first definition without reading the whole specification.

---

## Repository

- **GitHub**: [iblai/standard](https://github.com/iblai/standard)
- **License**: MIT

---

## Getting Started

```bash
git clone https://github.com/iblai/standard.git
cd standard
```

Read the specification to understand the `.iblai` file format, or run the generator to scaffold your first definition interactively. Existing agent rosters in the [claws](https://github.com/iblai/claws) repository are a useful reference for how the sections look in production.

---

## Related

- [Claw Agents](/developer/agents/claw-agents) — production agent rosters written against this format
- [Claw Setup](/developer/agents/claw-setup) — deploy agent definitions to a self-hosted server
