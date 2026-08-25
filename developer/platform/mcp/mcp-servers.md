# MCP Servers

[Model Context Protocol](https://modelcontextprotocol.io) servers and skills for operating the ibl.ai platform from your AI agent.

---

## Overview

**iblai/api** exposes ibl.ai platform capabilities as tools an AI agent can call, enabling deep integration between language models and the platform. It ships two things: a set of skills that teach your agent to drive the platform REST API directly, and a hosted MCP server for the one runtime capability that is not a REST call — chatting with a deployed agent.

Where [iblai/vibe](/developer/vibe) gives you components to *build* an app, this gives you the means to *run the platform itself*: configure agents, manage datasets and memory, administer users and roles, send notifications, and pull analytics for any organization you belong to.

Each capability maps to one skill and one set of exact REST endpoints — method, URL, body — so changing an agent's LLM or pulling cost analytics is a single command rather than a documentation hunt.

---

## Prerequisites

[Node.js](https://nodejs.org) for `npx`, a skills-compatible AI agent (Claude Code, Cursor, OpenCode, and others), and an ibl.ai account with an organization. Signing up at [ibl.ai/join](https://ibl.ai/join) creates both.

---

## Repository

- **GitHub**: [iblai/api](https://github.com/iblai/api)
- **License**: MIT

---

## Getting Started

Install the skills:

```bash
npx skills add iblai/api
```

Then run the login skill once. It reads your signed-in session, asks which organization to use, and writes your org key, username, and a Platform API Token to `.env`:

```text
/iblai-api-login
```

Every other skill reads `IBLAI_ORG`, `IBLAI_USERNAME`, and `IBLAI_API_KEY` from `.env` and calls the API with an `Authorization: Api-Token` header.

Running headless or in CI? Skip the browser — an org secret works directly as the API token. Set `IBLAI_ORG` to your org key and `IBLAI_API_KEY` to the org secret, then read `IBLAI_USERNAME` from the API.

These servers run against the hosted API environment. For a license to the full platform codebase, [contact the team](https://ibl.ai/contact).

---

## Related

- [App CLI](/developer/vibe) — scaffold an application on the platform
- [API Reference](/developer) — the underlying REST endpoints
