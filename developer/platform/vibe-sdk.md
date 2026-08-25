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

Open `http://localhost:3000`. You will be redirected to iblai.app for login -- sign in or create a free account, and you are back in your app with a fully authenticated session.

---

## What is Vibe

A developer toolkit for vibe coding with the ibl.ai platform. Vibe gives you a production-ready scaffold powered by the [@iblai/cli](https://github.com/iblai/vibe) command line tool, the [@iblai/iblai-js](https://www.npmjs.com/package/@iblai/iblai-js) SDK, pre-built components, Claude Code skills, and a full backend at iblai.app. You go from zero to a deployed AI app in minutes -- authentication, AI chat, profiles, notification, and analytics are already wired up. No API tokens to manage.

**Why it matters:**

- **Start building in minutes, not days** -- the CLI scaffolds a complete app with auth, AI chat, and a dashboard out of the box
- **Backend included** -- iblai.app provides SSO auth, AI agent infrastructure, analytics, and tenant management (free tier available)
- **Client-side auth via SSO** -- no API tokens to store, rotate, or leak
- **Claude Code skills guide every step** -- adding features is a conversation, not a scavenger hunt through docs
- **shadcn/ui fills in UI gaps** -- consistent design language without the overhead of a custom design system
- **Ship everywhere** -- web (platform hosting, your own container, or a static host), desktop (macOS/Windows/Linux), and mobile (iOS/Android) via Tauri v2

---

## How It Works

1. **Scaffold** -- run `npx @iblai/cli startapp agent` to generate a full Next.js app with auth, AI chat, profiles, and more
2. **Connect** -- your app connects to iblai.app (or your own tenant) for authentication, AI agents, and data
3. **Customize** -- use Claude Code skills to add features, swap components, and adjust business logic
4. **Deploy** -- ship through the platform's hosting API, run it in your own container, or package it with Tauri

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

After installing the skills, use them directly in your AI agent with `/` commands.

### Project and operations

| Skill | Description |
|-------|-------------|
| `/iblai-vibe-ops-init` | Start a new ibl.ai project and write the project CLAUDE.md |
| `/iblai-vibe-scaffold` | Scaffold a new app, or add ibl.ai features to an existing Next.js project |
| `/iblai-vibe-component` | Add an ibl.ai component or feature to your app |
| `/iblai-vibe-ops-test` | Add Vitest/Playwright tests and run the verification pass before showing work |
| `/iblai-vibe-ops-build` | Build and run on desktop and mobile (iOS, Android, macOS, Windows) |
| `/iblai-vibe-ops-deploy` | Deploy to ibl.ai hosting, a container, or a static host |
| `/iblai-vibe-ops-release` | Generate the Makefile and Fastlane config to submit to the App Store and Google Play |
| `/iblai-vibe-ops-upgrade` | Upgrade the `@iblai/iblai-js` SDK and the vibe skills to the latest versions |
| `/iblai-vibe-windows-msix` | Package a Tauri build as a Windows MSIX for sideloading or the Microsoft Store |
| `/iblai-vibe-iconography` | Generate every app-icon size for desktop and mobile builds from one source image |
| `/iblai-vibe-readme` | Write or refresh the README |

### App features

| Skill | Description |
|-------|-------------|
| `/iblai-vibe-auth` | Add ibl.ai SSO authentication to a vanilla Next.js app |
| `/iblai-vibe-agent-chat` | Add the in-process Chat surface (message stream, canvas, file attach, voice, prompts) |
| `/iblai-vibe-agent-chat-sidebar` | Wrap Chat with the SDK sidebar -- projects, pinned and recent messages |
| `/iblai-vibe-project` | Add the Projects surface (chat input, project files, instructions, assigned agents) |
| `/iblai-vibe-profile` | Add the profile dropdown and settings page |
| `/iblai-vibe-account` | Add the account and organization settings page |
| `/iblai-vibe-navbar` | Add a responsive navbar with logo, links, notification bell, and profile dropdown |
| `/iblai-vibe-notification` | Add the notification bell and notification center page |
| `/iblai-vibe-analytics` | Add the analytics dashboard page |
| `/iblai-vibe-invite` | Add user invitation dialogs |
| `/iblai-vibe-workflow` | Add workflow builder components |
| `/iblai-vibe-onboard` | Design and build a high-converting onboarding questionnaire flow |
| `/iblai-vibe-credit` | Add the ibl.ai credit balance widget |
| `/iblai-vibe-billing` | Add the tenant Billing surface (plan and credits, workspace spend limit, agent limits) |
| `/iblai-vibe-memory` | Add the tenant Memory admin surface (every user's global memories, every agent's memories) |
| `/iblai-vibe-history` | Add the profile History surface (a user's own conversations, with filters and exports) |
| `/iblai-vibe-local-llm` | Wire on-device LLM inference (Ollama) into a Next.js + Tauri desktop build |
| `/iblai-vibe-rbac` | Build and audit role-based access control -- roles, policies, action definitions |
| `/iblai-vibe-credential` | RBAC setup that lets a token list and unmask integration credentials |
| `/iblai-vibe-course-access` | Add course-content pages (edX user UI) |
| `/iblai-vibe-course-create` | Drive the Course Creation API end to end -- outline, units, review, publish |
| `/iblai-vibe-crm-overview` | Reference and index for the Platform-scoped CRM API and its workflow skills |

### Agent tabs

| Skill | Description |
|-------|-------------|
| `/iblai-vibe-agent-search` | Add the agent search/browse page (starred, featured, custom, default) |
| `/iblai-vibe-agent-setting` | Add the agent Settings tab (name, description, visibility, copy, delete) |
| `/iblai-vibe-agent-access` | Add the agent Access tab (RBAC for editor and chat roles) |
| `/iblai-vibe-agent-api` | Add the agent API tab (API key management) |
| `/iblai-vibe-agent-audit` | Add the agent Audit tab (who changed what and when, with filters) |
| `/iblai-vibe-agent-dataset` | Add the agent Datasets tab (searchable dataset table with upload) |
| `/iblai-vibe-agent-disclaimer` | Add the agent Disclaimers tab (user agreement and advisory) |
| `/iblai-vibe-agent-embed` | Add the agent Embed tab (embed code, custom styling, shareable links) |
| `/iblai-vibe-agent-evals` | Add the agent Evals tab (benchmarks, LLM-as-Judge reviews, manual scores, CSV export) |
| `/iblai-vibe-agent-grader` | Add the agent Grader tab (rubric grading, criteria table, results with LMS-synced overrides) |
| `/iblai-vibe-agent-history` | Add the agent History tab (conversation history with filters and export) |
| `/iblai-vibe-agent-llm` | Add the agent LLM tab (model provider selection) |
| `/iblai-vibe-agent-lti` | Add the agent LTI tab (LTI 1.3 launch toggle, links, signing keys, tools, endpoints) |
| `/iblai-vibe-agent-mcp` | Add the agent MCP tab (featured and custom connectors, OAuth, add/edit dialogs) |
| `/iblai-vibe-agent-memory` | Add the agent Memory tab (enable/disable memory and manage memories) |
| `/iblai-vibe-agent-privacy` | Add the agent Privacy tab (PII detection with redact/mask/block actions) |
| `/iblai-vibe-agent-prompt` | Add the agent Prompts tab (system prompts and suggested prompts) |
| `/iblai-vibe-agent-safety` | Add the agent Safety tab (moderation prompts and flagged content) |
| `/iblai-vibe-agent-sandbox` | Add the agent Sandbox tab (Claw instance management and agent prompt configuration) |
| `/iblai-vibe-agent-skills` | Add the agent Skills tab (Agent Skills catalog, per-agent assignment, private skills, file resources, and the chat `/` skill picker) |
| `/iblai-vibe-agent-task` | Add the agent Tasks tab (schedule automated periodic tasks with run logs) |
| `/iblai-vibe-agent-tool` | Add the agent Tools tab (enable/disable agent tools) |
| `/iblai-vibe-agent-support` | Add the agent Support tab (ticket inbox, availability toggle, filters, replies) |
| `/iblai-vibe-agent-voice` | Add the agent Voice tab (voice source, voice picker, voice instructions, and call configuration) |
| `/iblai-vibe-agent-billing` | Add the agent Billing tab (per-agent and per-user LLM spend caps with usage bars and enforcement) |

### Monetization

| Skill | Description |
|-------|-------------|
| `/iblai-vibe-monetization` | Reference and index for item-level monetization (Stripe Connect, paywalls, tiers) |
| `/iblai-vibe-monetization-onboard` | Build the Stripe Connect Express onboarding surface and payment-readiness gate |
| `/iblai-vibe-monetization-configure` | Build the admin monetization tab -- paywall settings and per-item pricing tiers |
| `/iblai-vibe-monetization-checkout` | Build the paywall modal, access-check gate, and Stripe checkout (including guest buy) |
| `/iblai-vibe-monetization-subscription` | Build the user Purchases tab -- list, detail, and cancel |
| `/iblai-vibe-monetization-analytics` | Build revenue dashboards, subscriber lists, and paywall overviews |
| `/iblai-vibe-monetization-app-paywall` | Gate a whole app behind Stripe using the tenant's own key |

### Design and code quality

| Skill | Description |
|-------|-------------|
| `/iblai-vibe-design` | Design, critique, and polish frontend interfaces against the ibl.ai design system |
| `/iblai-vibe-deslop` | Audit and harden an existing codebase for production readiness |

### Security

| Skill | Description |
|-------|-------------|
| `/iblai-vibe-security-owasp-audit` | Audit application source against the OWASP Top 10 |
| `/iblai-vibe-security-dependency-audit` | Audit dependencies and frameworks for known CVEs and supply-chain risk |
| `/iblai-vibe-security-cloud-audit` | Audit AWS, GCP, and Azure for misconfigurations and excessive permissions |
| `/iblai-vibe-security-prompt-injection` | Audit AI features for prompt injection and agent permission-boundary flaws |
| `/iblai-vibe-security-incident-triage` | Triage a security incident following NIST SP 800-61 |
| `/iblai-vibe-security-recon` | Structured reconnaissance and attack-surface enumeration for authorized tests |
| `/iblai-vibe-security-osint-recon` | Correlate open source intelligence for authorized investigations |
| `/iblai-vibe-security-disk-forensics` | Analyze disk images and file systems for evidence recovery |

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

### Platform hosting (default)

`/iblai-vibe-ops-deploy` ships the app through the ibl.ai platform's own hosting API. The platform holds the hosting credential for your tenant, so the only configuration you need is the `DOMAIN`, `PLATFORM`, and `TOKEN` already in `iblai.env` — **no third-party hosting account, token, or CLI**. The skill zips the app, posts it to the hosting endpoint, and polls until the build is ready; redeploying is the same call with the same project slug.

### Your own infrastructure

When the app has to run where you control it — your own server, on-premise, Kubernetes, Cloud Run, or air-gapped — deploy it as a container. The same skill covers this path, and a project that already carries a `Dockerfile` deploys the way it already deploys.

```bash
docker build -t my-vibe-app .
docker run -p 3000:3000 my-vibe-app
```

A static host is the third option, for an app with no server-side runtime at all.

Where deployment is a data-residency or compliance decision rather than a convenience one, the choice belongs to the organization — the tooling supports all three targets equally.

### Tauri (Desktop & Mobile)

Build native apps for macOS, Windows, Linux, iOS, and Android:

```bash
iblai add builds              # Add Tauri support
iblai builds build            # Desktop build for current platform
iblai builds ios init         # iOS project setup
iblai builds ci-workflow --all  # GitHub Actions for all platforms
```

`/iblai-vibe-ops-build` covers signed and notarized macOS and Windows release builds; `/iblai-vibe-ops-release` generates the Makefile and Fastlane configuration for submitting to the Apple App Store and Google Play, and `/iblai-vibe-windows-msix` packages a build as a Windows MSIX for sideloading or the Microsoft Store.

---

## Resources

- [@iblai/cli](https://github.com/iblai/vibe) -- the CLI that scaffolds Vibe apps
- [@iblai/iblai-js](https://www.npmjs.com/package/@iblai/iblai-js) -- unified SDK for data, UI components, and auth utilities
- [@iblai/iblai-api](https://www.npmjs.com/package/@iblai/iblai-api) -- auto-generated API types
- [@iblai/mcp](https://www.npmjs.com/package/@iblai/mcp) -- MCP server for AI-assisted development
- [skills.sh/iblai/vibe](https://skills.sh/iblai/vibe) -- install skills with `npx skills add iblai/vibe`
