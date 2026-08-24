# OS Platform Overview

![The OS chat interface at os.ibl.ai, showing the agent workspace with the message composer and sidebar](/images/docs/os/chat-canvas/chat.webp)

## What Is OS?

OS is ibl.ai's open-source AI agent platform. It lets organizations build, deploy, and manage intelligent conversational agents — from prototype to production — on infrastructure where you own all the code and the data, with full control over models and branding.

The platform runs in production at [os.ibl.ai](https://os.ibl.ai) and the source code is available at [github.com/iblai/os](https://github.com/iblai/os). It ships as a web application, as desktop apps for macOS, Windows, and Linux, and as mobile apps for iOS and Android.

This documentation section covers every screen of the application: the chat and Canvas workspace, per-agent settings, analytics dashboards, organization administration, user profiles, and notifications.

## Core Capabilities

#### AI Agents
Create custom agents with configurable LLMs, system prompts, tools, and safety filters. Every agent has its own settings, datasets, and access rules — see [Agent Settings: Basic Information](agent-settings/basic.md).

#### RAG Training
Ground agents in your own data: upload documents, connect Google Drive, OneDrive, or Dropbox, or crawl websites. Managed per agent under [Datasets](agent-settings/datasets.md) and per organization under [Data Source Integrations](organization-settings/integrations-data-sources.md).

#### Voice Calls
Real-time voice conversations with agents over WebRTC, powered by LiveKit. Configured in [Voice settings](agent-settings/voice.md) and [Voice Calls](agent-settings/voice-call.md).

#### Canvas / Artifacts
Generate, edit, version, and export rich documents alongside chat. See [Canvas](chat-canvas/canvas.md), [Canvas Options](chat-canvas/canvas-options.md), and [Downloading a Canvas](chat-canvas/canvas-download.md).

#### Deep Research & Web Search
Extended multi-step reasoning for complex queries, plus live web results to ground responses. Available from the [chat interface](chat-canvas/chat.md).

#### MCP Servers
Extend agent capabilities with Model Context Protocol tool servers, configured per agent in [MCP Servers](agent-settings/mcp.md).

#### Analytics
Usage dashboards, per-user activity, topic analysis, a transcript viewer, cost tracking, and exportable data reports. See the [Analytics section](analytics/overview.md).

#### Evaluations
Measure and improve agent quality: run an agent against a benchmark of test questions, then score every response with LLM-as-Judge reviews and human annotations, and export the results. See [Evals](agent-settings/evals.md).

#### Rubric Grading
Let an agent grade work against a rubric you define: choose whether it scores a submission or the whole conversation, how much feedback the person sees, and the criteria and point values it marks against. See [Grader](agent-settings/grader.md).

#### Projects & Workflows
Group a body of work — its files, its standing instructions, and the agents assigned to it — into a [Project](chat-canvas/projects.md), and lay out a multi-step agent run as a graph in the [workflow builder](chat-canvas/workflows.md), with branches, loops, guardrail checks, and human approval steps.

#### Multi-Organization, SSO & RBAC
Full organization isolation with per-organization configuration and branding, Single Sign-On, and granular role-based access control with roles, policies, groups, and teams. See [Organization Settings](organization-settings/organization.md).

#### Billing, Spend Limits & Monetization
Stripe-backed subscription management and usage-based pricing at the organization level, plus enforceable ceilings on LLM spend at three scopes — the whole workspace, one agent, or one person on one agent — that block or alert when they are reached. See [Billing](organization-settings/billing.md), [Agent Billing](agent-settings/billing.md), and [Monetization](organization-settings/monetization.md).

#### Memory Administration
Memories belong to people and to agents, and an administrator can manage both: a user's own memories on their [Profile](profile/memory.md), an agent's in its [Memory settings](agent-settings/memory.md), and everyone's from the organization's [Memory](organization-settings/memory.md) tab — including the switches that decide whether anything is captured at all.

#### Conversation History & Export
Every user can review and export their own conversations from [Profile: History](profile/history.md), while administrators review a whole agent's conversations in [Chat History](agent-settings/history.md).

#### Embedding & API Access
Embed agents in any website via iframe with custom styling, and integrate programmatically with API keys and LTI 1.3 for LMS platforms. See [Embed](agent-settings/embed.md), [API Access](agent-settings/api.md), and the LTI pages.

#### Human Support
Let people chatting with an agent hand the conversation to a human. Escalations arrive as tickets a support team can filter, reply to, and resolve — in the app or over the API. See [Support](agent-settings/support.md).

## Documentation Map

#### Chat & Canvas
The end-user workspace: [Chat Interface](chat-canvas/chat.md) · [Explore Agents](chat-canvas/explore.md) · [Projects](chat-canvas/projects.md) · [Workflows](chat-canvas/workflows.md) · [Canvas](chat-canvas/canvas.md) · [Canvas Options](chat-canvas/canvas-options.md) · [Downloading a Canvas](chat-canvas/canvas-download.md) · [Renaming a Canvas](chat-canvas/canvas-rename.md)

#### Agent Settings
Everything configurable on a single agent: [Basic](agent-settings/basic.md) · [System Prompt](agent-settings/prompt.md) · [LLM Selection](agent-settings/llm-selection.md) · [LLM Configuration](agent-settings/llm-configuration.md) · [Capabilities](agent-settings/capabilities.md) · [Grader](agent-settings/grader.md) · [Tools](agent-settings/tools.md) · [Skills](agent-settings/skills.md) · [MCP Servers](agent-settings/mcp.md) · [Datasets](agent-settings/datasets.md) · [Memory](agent-settings/memory.md) · [Voice](agent-settings/voice.md) · [Voice Selector](agent-settings/voice-selector.md) · [Voice Calls](agent-settings/voice-call.md) · [Safety](agent-settings/safety.md) · [Privacy](agent-settings/privacy.md) · [Disclaimers](agent-settings/disclaimers.md) · [Discovery](agent-settings/discovery.md) · [Access Control](agent-settings/access.md) · [Embed](agent-settings/embed.md) · [API Access](agent-settings/api.md) · [Agent Analytics](agent-settings/analytics.md) · [Evals](agent-settings/evals.md) · [Audit Log](agent-settings/audit.md) · [Chat History](agent-settings/history.md) · [Support](agent-settings/support.md) · [Tasks](agent-settings/tasks.md) · [Billing](agent-settings/billing.md) · [Sandbox](agent-settings/sandbox.md) · [LTI Keys](agent-settings/lti-keys.md) · [LTI Links](agent-settings/lti-links.md) · [LTI Tool Endpoints](agent-settings/lti-tool-endpoints.md) · [LTI Tools](agent-settings/lti-tools.md)

#### Analytics
Organization-wide insight: [Overview Dashboard](analytics/overview.md) · [Users](analytics/users.md) · [Topics](analytics/topics.md) · [Transcripts](analytics/transcripts.md) · [Costs](analytics/costs.md) · [Data Reports](analytics/data-reports.md) · [Audit](analytics/audit.md)

#### Organization Settings
Organization administration: [Organization](organization-settings/organization.md) · [Users](organization-settings/users.md) · [Groups](organization-settings/groups.md) · [Teams](organization-settings/teams.md) · [Roles](organization-settings/roles.md) · [Policies](organization-settings/policies.md) · [Alerts](organization-settings/alerts.md) · [LLM Integrations](organization-settings/integrations-llms.md) · [Data Source Integrations](organization-settings/integrations-data-sources.md) · [Memory](organization-settings/memory.md) · [Billing](organization-settings/billing.md) · [Monetization](organization-settings/monetization.md) · [Advanced](organization-settings/advanced.md)

#### Profile
Per-user account pages: [Basic Information](profile/basic.md) · [Education](profile/education.md) · [Education Credentials](profile/education-credentials.md) · [Experience](profile/experience.md) · [Resume](profile/experience-resume.md) · [Social Links](profile/social.md) · [Memory](profile/memory.md) · [History](profile/history.md) · [Privacy](profile/privacy.md) · [Security](profile/security.md) · [Purchases](profile/purchases.md)

#### Notifications
Staying informed: [Inbox](notifications/inbox.md) · [Alerts](notifications/alerts.md)

#### Common Components
Shared building blocks that appear across all ibl.ai applications — the profile dialog, notification center, analytics dashboards, and organization settings: [Common Components](common-components/overview.md)

## Related Resources

#### Source Code
The full application is open source at [github.com/iblai/os](https://github.com/iblai/os) under the MIT license.

#### API Skills & MCP Server
[github.com/iblai/api](https://github.com/iblai/api) packages 33 agent skills and a chat MCP server for operating any ibl.ai organization from an AI agent.

#### Vibe
[github.com/iblai/vibe](https://github.com/iblai/vibe) is the companion toolkit for building and shipping ibl.ai applications, including the build skills used to produce the desktop and mobile releases of OS.
