# Vibe

Ship AI-powered apps fast. Backend included.

> Source: [github.com/iblai/vibe](https://github.com/iblai/vibe)

**Developer**

---

## Quick Start

### Install Skills

Add ibl.ai skills to any project with one command:

```bash
npx skills add iblai/vibe
```

### ibl.ai App Template

Get a complete app with auth, AI chat, profiles, and more in one command:

```bash
npx @iblai/cli startapp agent -o iblai-init
cp -a iblai-init/<app-name>/. . && rm -rf iblai-init
rm -rf node_modules && pnpm install
cp .env.example .env.local
pnpm dev
```

Open http://localhost:3000. You will be redirected to iblai.app for login -- sign in or create a free account, and you are back in your app with a fully authenticated session.

---

## What is Vibe

A developer toolkit for vibe coding with the ibl.ai platform. Vibe gives you a production-ready scaffold powered by [iblai-app-cli](https://github.com/iblai/iblai-app-cli), the [@iblai/iblai-js](https://www.npmjs.com/package/@iblai/iblai-js) SDK, pre-built components, Claude Code skills, and a full backend at iblai.app. You go from zero to a deployed AI app in minutes -- authentication, AI chat, profiles, notification, and analytics are already wired up. No API tokens to manage.

**Why it matters:**

- **Start building in minutes, not days** -- the CLI scaffolds a complete app with auth, AI chat, and a dashboard out of the box
- **Backend included** -- iblai.app provides SSO auth, AI agent infrastructure, analytics, and tenant management (free tier available)
- **Client-side auth via SSO** -- no API tokens to store, rotate, or leak
- **Claude Code skills guide every step** -- adding features is a conversation, not a scavenger hunt through docs
- **shadcn/ui fills in UI gaps** -- consistent design language without the overhead of a custom design system
- **Ship everywhere** -- web (Vercel), desktop (macOS/Windows/Linux), and mobile (iOS/Android) via Tauri v2

---

## How It Works

1. **Scaffold** -- run `npx @iblai/cli startapp agent` to generate a full Next.js app with auth, AI chat, profiles, and more
2. **Connect** -- your app connects to iblai.app (or your own tenant) for authentication, AI agents, and data
3. **Customize** -- use Claude Code skills to add features, swap components, and adjust business logic
4. **Deploy** -- push to Vercel, package with Tauri, or run in Docker

---

## What You Get

| Feature | Description |
|---------|-------------|
| **Authentication** | SSO login via iblai.app -- no token management, session handling built in |
| **AI Chat** | Streaming chat with ibl.ai agents, markdown rendering, conversation history |
| **User Profile** | Editable profile page with avatar, bio, and preferences |
| **Account Settings** | Password changes, notification preferences, connected services |
| **Analytics Dashboard** | Usage metrics, conversation stats, and user activity |
| **Notification** | Real-time notification system with read/unread state |
| **Desktop & Mobile** | Tauri v2 integration for macOS, Windows, Linux, iOS, and Android |
| **AI Development Skills** | Claude Code skills that walk you through adding and customizing every feature |

---

## Skills

After installing the skills, use them directly in your AI agent with `/` commands:

| Skill | Description |
|-------|-------------|
| `/iblai-auth` | Add SSO authentication (includes CLI installation guide) |
| `/iblai-chat` | Add AI chat widget |
| `/iblai-profile` | Add profile dropdown + settings page |
| `/iblai-account` | Add account/org settings page |
| `/iblai-analytics` | Add analytics dashboard |
| `/iblai-notification` | Add notification bell + center page |
| `/iblai-invite` | Add user invitation dialogs |
| `/iblai-workflow` | Add workflow builder components |
| `/iblai-onboard` | Design and build a high-converting onboarding questionnaire flow |
| `/iblai-build` | Build and run on desktop and mobile (iOS, Android, macOS, Windows) |
| `/iblai-screenshot` | Capture app store screenshots for web, iOS, and Android |
| `/iblai-test` | Test your app before showing work to the user |
| `/iblai-agent-search` | Add the agent search/browse page (starred, featured, custom, default) |
| `/iblai-agent-settings` | Add the agent Settings tab (name, visibility, copy, delete) |
| `/iblai-agent-access` | Add the agent Access tab (RBAC for editor and chat roles) |
| `/iblai-agent-api` | Add the agent API tab (API key management) |
| `/iblai-agent-datasets` | Add the agent Datasets tab (searchable dataset table with upload) |
| `/iblai-agent-disclaimers` | Add the agent Disclaimers tab (user agreement and advisory) |
| `/iblai-agent-embed` | Add the agent Embed tab (embed code, custom styling, shareable links) |
| `/iblai-agent-history` | Add the agent History tab (conversation history with filters and export) |
| `/iblai-agent-llm` | Add the agent LLM tab (model provider selection) |
| `/iblai-agent-memory` | Add the agent Memory tab (enable/disable memory and manage memories) |
| `/iblai-agent-prompts` | Add the agent Prompts tab (system prompts and suggested prompts) |
| `/iblai-agent-safety` | Add the agent Safety tab (moderation prompts and flagged content) |
| `/iblai-agent-tools` | Add the agent Tools tab (enable/disable agent tools) |

Skills are in `skills/` (symlinked to `.claude/skills/`). Read them, extend them, or write your own.

---

## Add to Existing Apps

Already have a project? Install the skills and let your AI agent add features:

```bash
npx skills add iblai/vibe
```

Then use the CLI to add features:

```bash
iblai add mcp            # MCP servers + skills (run first)
iblai add auth           # SSO authentication
iblai add chat           # AI chat with streaming
iblai add profile        # User profile dropdown
iblai add account        # Account/organization settings
iblai add analytics      # Analytics dashboard
iblai add notification   # Notification bell
```

### CI/CD

Use `--yes` to skip interactive prompts:

```bash
npx @iblai/cli startapp agent --yes --platform acme --agent my-id --app-name my-app -o iblai-init
cp -a iblai-init/my-app/. . && rm -rf iblai-init
rm -rf node_modules && pnpm install
cp .env.example .env.local
```

---

## The iblai Backend

iblai.app is the production backend that powers every Vibe app. You do not need to build, host, or maintain any backend services.

**What iblai.app provides:**

- **SSO Authentication** -- OAuth-based login with session management, RBAC, and multi-tenant user isolation
- **AI Agent Infrastructure** -- create, configure, and serve AI agents with streaming responses, tool use, and RAG
- **Analytics** -- track user activity, conversation metrics, and engagement across your app
- **Tenant Management** -- each tenant gets its own users, agents, branding, and configuration

---

## AI-Assisted Development

Vibe is designed to be built with AI. The [@iblai/mcp](https://www.npmjs.com/package/@iblai/mcp) server gives Claude Code deep knowledge of the ibl.ai platform, and the bundled skills guide you through every common task.

### MCP Server

Add this to your `.mcp.json` at the project root:

```json
{
  "mcpServers": {
    "iblai": {
      "command": "npx",
      "args": ["-y", "@iblai/mcp"]
    }
  }
}
```

This gives your AI assistant access to:

```
get_component_info("ChatWidget")              # Props, usage, examples for any component
get_hook_info("useAdvancedChat")              # Hook parameters and return types
get_api_query_info("useGetUserMetadataQuery") # RTK Query endpoint details
get_provider_setup("auth")                    # Provider hierarchy and setup code
create_page_template("Dashboard", "agent")   # Generate a page following ibl.ai patterns
```

---

## Platform Capabilities

| Feature | Web | macOS | Windows/Surface | iOS | Android |
|---------|-----|-------|-----------------|-----|---------|
| SSO Authentication | Yes | Yes | Yes | No | No |
| AI Chat | Yes | Yes | Yes | Yes | Yes |
| User Profile | Yes | Yes | Yes | Yes | Yes |
| Account Settings | Yes | Yes | Yes | Yes | Yes |
| Analytics Dashboard | Yes | Yes | Yes | Yes | Yes |
| Notifications | Yes | Yes | Yes | Yes | Yes |

> **iOS & Android SSO limitation:** Mobile WebViews use a non-standard user-agent that SSO providers reject. Completing the OAuth flow requires a system browser popup (ASWebAuthenticationSession on iOS, Chrome Custom Tabs on Android). This is not yet implemented -- mobile users must authenticate via another method for now.

---

## Deploy Anywhere

### Vercel (recommended)

One-click deploy. Connect your repo, set your environment variables, and push.

```bash
vercel --prod
```

### Docker

```bash
docker build -t my-vibe-app .
docker run -p 3000:3000 my-vibe-app
```

### Tauri (Desktop & Mobile)

Build native apps for macOS, Windows, Linux, iOS, and Android:

```bash
iblai add builds              # Add Tauri support
iblai builds build            # Desktop build for current platform
iblai builds ios init         # iOS project setup
iblai builds ci-workflow --all  # GitHub Actions for all platforms
```

---

## Resources

- [iblai-app-cli](https://github.com/iblai/iblai-app-cli) -- the CLI that scaffolds Vibe apps
- [@iblai/iblai-js](https://www.npmjs.com/package/@iblai/iblai-js) -- unified SDK for data, UI components, and auth utilities
- [@iblai/iblai-api](https://www.npmjs.com/package/@iblai/iblai-api) -- auto-generated API types
- [@iblai/mcp](https://www.npmjs.com/package/@iblai/mcp) -- MCP server for AI-assisted development
- [skills.sh/iblai/vibe](https://skills.sh/iblai/vibe) -- install skills with `npx skills add iblai/vibe`
