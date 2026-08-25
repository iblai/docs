# .iblai Agent File Format

> Mirrored from [`iblai/standard`](https://github.com/iblai/standard) · [`README.md`](https://github.com/iblai/standard/blob/main/README.md). This page is generated — edit it in the repository, not here.

A Markdown-based portable agent definition format. One file captures everything needed to define, share, and load an AI agent — across tools, teams, and platforms.

  [standard v1.0.0](https://github.com/iblai/standard/releases/tag/v1.0.0)
  [MIT License](https://github.com/iblai/standard/blob/main/LICENSE)
  [.iblai format](https://github.com/iblai/standard)

```
┌  Let's create your agent  🚀
│
  ██░░░░░░░░░░░░░░░░░░░░░░ 7% (1/15)

  ┌──────────────────────────────────────────────┐
  │  🤖  Agent Info                              │
  │     Name, description, and metadata          │
  └──────────────────────────────────────────────┘

│  ◆  Agent name
│  │  Campus Advisor
│
│  ◆  Description
│  │  Academic advisor for undergrad students
│
│  ◇  Version
│  │  1.0.0
│
  ✔ Setting up identity...

│  ✦ Configure Soul section?
│  │  Yes
│
  ███░░░░░░░░░░░░░░░░░░░░░ 13% (2/15)

  ┌──────────────────────────────────────────────┐
  │  🧠  Soul                                    │
  │     Voice, temperament, values, ethics       │
  └──────────────────────────────────────────────┘

│  ◆  Voice — describe tone, register, language style
│  │  Warm, encouraging, and approachable.
│
│  ✦ Add values?
│  │  Yes
│
│  ◆  Enter a core value
│  │  Student success comes first
│  ✔  › Student success comes first
│
  ✔ Shaping personality...

  ...

  ██████████████░░░░░░░░░░ 60% (9/15)

  ┌──────────────────────────────────────────────┐
  │  ⚡  Skills                                   │
  │     Discrete capabilities your agent has     │
  └──────────────────────────────────────────────┘

│  ⚡ Skill #1
│  ◆  Skill name
│  │  Degree Audit
│  ✔ Skill "Degree Audit" added
│
  ✔ Wiring up skills...

  ...

  ████████████████████████ 100% (15/15)

  🎉 All sections complete!

  ┌──────────────────────────────────────────────┐
  │  📄  Output                                  │
  │     Save your new .iblai file                │
  └──────────────────────────────────────────────┘

│  ◆  File path for the .iblai file
│  │  campus-advisor.iblai
│
  ✔ Writing file...

└  ✨ Agent file written to ./campus-advisor.iblai
```

---

## File Structure

```
standard/
├── README.md
├── LICENSE
├── .gitignore
├── cli/                      # Interactive agent builder wizard
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts          # CLI entry, routing, help screen
│       ├── agent.ts          # `iblai agent` command
│       ├── theme.ts          # ibl.ai color palette & animations
│       ├── logo.ts           # ASCII art gradient logo
│       ├── wizard.ts         # Wizard step orchestration
│       └── generator.ts      # Data → .iblai Markdown
└── library/
    ├── university-advisor.iblai
    ├── customer-service.iblai
    ├── it-helpdesk.iblai
    └── .secrets              # Placeholder secrets template
```

## Quick Start

1. Pick an example from `library/` or start from scratch.
2. Copy `.secrets` alongside it and fill in real values.
3. Edit each section to match your agent.
4. Load the file into your platform of choice.

## Sections

An `.iblai` file is standard Markdown organized by headings. Every section is optional — include only what your agent needs.

### Agent Info

Core metadata for the agent. Fields: `name`, `description`, `picture` (URL or file path to an avatar), `version`, and `author`.

### Soul

Defines who the agent is at its core. Contains subsections for **Voice** (tone, register, language style), **Temperament** (how the agent handles ambiguity, pressure, and unknowns), **Values** (ranked principles that guide decision-making), and **Ethical Constraints** (hard boundaries the agent will never cross).

### Identity

Controls how the agent presents itself visually and socially. Fields: `emoji` (a single emoji used as the agent's icon), `theme` (light or dark), and `persona` (a one-line description of the agent's character).

### Instructions

The operating contract that governs how the agent works. Contains subsections for **Priorities** (an ordered list of what matters most), **Boundaries** (what the agent must not do), **Workflow** (step-by-step process the agent follows for each request), and **Quality Bar** (minimum standards every response must meet).

### User Preferences

Captures what the user expects from every interaction. Fields: `tone` (e.g. professional, casual, academic), `output_format` (e.g. markdown, plain text, JSON), `language` (ISO language code), and `constraints` (recurring rules like word limits or jargon restrictions).

### Memory

Long-lived persistent facts the agent retains across sessions. Use this section to store durable context like project stack details, deployment targets, team conventions, and past decisions. The agent should treat these as ground truth until explicitly updated.

### Heartbeat

Periodic maintenance routines the agent runs on a schedule. Fields: `cadence` (e.g. daily, weekly, hourly) and `tasks` (a list of recurring actions like reviewing open issues, checking for dependency updates, or running health checks).

### Skills

Discrete, reusable capabilities the agent can perform. Each skill is defined under its own subheading with fields: `id` (unique identifier), `description` (what the skill does), `tags` (categories for discovery), and `examples` (sample user prompts that trigger the skill).

### Tasks

Scheduled jobs the agent runs automatically. Each task is defined under its own subheading with fields: `frequency` (either cron notation like `0 9 * * 1` or a natural-language schedule like "every Monday at 9am") and `prompt` (the exact instruction sent to the agent when the task fires, written as a user message).

### Tools

Toggle-able built-in tools. Set each to `true` or `false`. Available tools: `web_search`, `code_interpreter`, `file_search`, `computer_use`, `shell`, `image_generation`, `url_context`, `maps`, `deep_research`, `function_calling`.

### MCP Servers

Model Context Protocol server connections. Each server is defined under its own subheading with fields: `name`, `url`, and `transport` (e.g. streamable-http, stdio). No secrets should appear in this section.

### Secrets

A reference to the companion secrets file. Contains a single field: `secrets_file` (path to a `.env` file). No actual secrets should ever appear in the `.iblai` file itself.

### Knowledge & RAG

References to datasets the agent uses for retrieval-augmented generation. List URLs, local file paths, or folder paths under **Sources**. The platform will index these and make them available to the agent at query time.

### Guardrails & Safety

Safety controls applied to every interaction. Contains subsections for **Input Filters** (preprocessing rules like PII stripping or length limits), **Output Filters** (postprocessing rules like redaction or disclaimer injection), **Content Policies** (categories of content the agent must never produce), **Blocked Topics** (specific subjects the agent must refuse), and **Prompt Safety Layers** (system-level instructions reinforcing safety behavior).

### Visibility & Permissions

Access control for the agent definition. Fields: `view` (`anyone` or `restricted`), `edit` (`anyone` or `restricted`), `editors` (list of emails or groups allowed to edit), `viewers` (list of emails or groups allowed to view).

## CLI — Agent Builder

Build `.iblai` files interactively from your terminal with an animated, step-by-step wizard.

### Quick start

```bash
cd cli
npm install
npm run build
npm link
```

Then from anywhere:

```bash
iblai agent        # Launch the interactive agent builder
iblai help         # Show available commands
```

> Once the package is published to npm, you'll be able to run `npx iblai agent` directly.

### What `iblai agent` does

Running `iblai agent` starts a guided wizard that walks you through all 15 sections of the `.iblai` format:

1. **Agent Info** — name, description, picture, version, author
2. **Soul** — voice, temperament, values, ethical constraints
3. **Identity** — emoji, theme, persona
4. **Instructions** — priorities, boundaries, workflow, quality bar
5. **User Preferences** — tone, output format, language, constraints
6. **Memory** — persistent facts
7. **Heartbeat** — cadence and recurring tasks
8. **Skills** — add multiple skills with IDs, tags, and examples
9. **Tasks** — scheduled jobs with cron or natural-language frequency
10. **Tools** — toggle built-in tools on/off
11. **MCP Servers** — connect external servers
12. **Secrets** — reference a secrets file
13. **Knowledge & RAG** — add data source URLs/paths
14. **Guardrails & Safety** — input/output filters, content policies, blocked topics
15. **Visibility & Permissions** — view/edit access control

Every section can be skipped. Loop-based sections (skills, tasks, MCP servers, etc.) let you add as many entries as you need. A progress bar tracks where you are. At the end, the wizard writes a ready-to-use `.iblai` file that matches the format in `library/`.

## Design Principles

- **Human-readable** — Plain Markdown; no custom parser required to understand the file.
- **Portable** — One file captures the full agent definition.
- **Secure** — Secrets live in a separate `.env` file that is gitignored by default.
- **Extensible** — Add custom sections as needed; unknown headings are ignored by conforming parsers.

## Inspiration

The `.iblai` format draws from:

- [OpenClaw](https://github.com/anthropics/openclaw) — Agent identity, soul, memory, skills, and heartbeat concepts.
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io) — Server connectivity for tools and context.
- Gemini + OpenAI SDKs — Union of built-in tool capabilities.

## License

MIT — see [LICENSE](https://github.com/iblai/standard/blob/main/LICENSE) for details.
