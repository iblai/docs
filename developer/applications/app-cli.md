# App CLI

Interactive CLI for scaffolding [ibl.ai frontend applications](https://github.com/iblai/vibe) with Next.js and React.

---

## Overview

**iblai-app-cli** is the scaffolding tool inside [iblai/vibe](https://github.com/iblai/vibe). It generates a production-ready Next.js and React application wired against the ibl.ai backend, so authentication, AI chat, profiles, notifications, and analytics are working before you write your first feature.

The scaffold is powered by the [@iblai/iblai-js](https://www.npmjs.com/package/@iblai/iblai-js) SDK and ships with pre-built components, configuration presets, and Claude Code skills that guide each step of extending the app.

---

## Why use it

- **Start in minutes, not days** — the starter scaffolds a complete app with auth, AI chat, and a dashboard out of the box
- **Backend included** — `iblai.app` supplies SSO auth, agent infrastructure, analytics, and tenant management, with a free tier
- **Client-side auth via SSO** — no API tokens to store, rotate, or leak
- **Skills guide every step** — adding a feature is a conversation with your coding agent rather than a hunt through docs
- **shadcn/ui fills UI gaps** — a consistent design language without maintaining a custom design system
- **Ship everywhere** — web on Vercel, plus desktop (macOS, Windows, Linux) and mobile (iOS, Android) via Tauri v2

---

## Repository

- **GitHub**: [iblai/vibe](https://github.com/iblai/vibe)
- **License**: MIT

---

## Getting Started

Install the skills into any project with a single command:

```bash
npx skills add iblai/vibe
```

Then ask your coding agent to start an ibl.ai agent app, or to add an ibl.ai Chat, Profile, Account, Notification, or Analytics component to an existing Next.js project. To work from the repository directly instead:

```bash
git clone https://github.com/iblai/vibe.git
cd vibe
```

The toolkit runs against the hosted `iblai.app` environment. For a license to the full platform codebase — to run it locally or self-host it — [contact the team](https://ibl.ai/contact).

---

## Related

- [Vibe](/developer/repos/vibe) — the full toolkit reference
- [MCP Servers](/developer/applications/mcp) — drive the platform API from your agent
