# MCP Connector Authentication

<iframe width="560" height="315" src="https://www.youtube.com/embed/fcFsqz7PndI" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Overview

Configure MCP connectors with different authentication methods and scopes to control how users authenticate with external services like Notion, Google, and other OAuth-based providers.

---

## Adding a Custom MCP Connector

1. Go to the **MCP** tab on an agent.
2. Click **Add Custom MCP Connector**.
3. Provide the **connector URL**.
4. Select the **connector scope**:
   - **This agent only**: the connector is available only for this agent
   - **All agents**: the connector is available for every agent in the tenant

---

## Authentication Methods

### API Key

- Select **API Key** as the authentication method.
- Choose the **token type** for the header (e.g., Bearer, Basic, Token).
- Select **Other** to provide a custom token type.
- Enter the **token value**.

### OAuth

For OAuth-based connectors (e.g., Notion), choose the **OAuth** authentication method and configure the **authentication scope**.

---

## OAuth Authentication Scopes

| Scope | Behavior |
|-------|----------|
| **Tenant** | Authenticate once — the authentication is shared across all tenant members and all agents |
| **Agent** | Authenticate once — the authentication applies to this agent only but is shared across all users |
| **User** | Each user must authenticate individually when they first interact with the agent |

---

## User-Based Authentication Flow

1. Admin adds the OAuth MCP connector with **User** scope.
2. Admin authenticates first (completes the OAuth flow).
3. The connector is enabled on the agent.
4. When a different user chats with the agent:
   - An authentication prompt appears: "Authentication required for [service]. Please complete the login in the open window."
   - The user completes the OAuth flow in a popup window.
   - Upon success, the chat continues with the connector active.

---

## Enable/Disable Connectors

- After adding a connector, use the **toggle** to enable or disable it.
- **Enabled**: the agent can use the MCP connector in responses.
- **Disabled**: the agent cannot access the connector.

---

## Key Takeaways

- **Tenant** scope: authenticate once, everyone benefits
- **Agent** scope: authenticate once per agent, shared across users
- **User** scope: every user authenticates individually (most secure)
- Connectors can be scoped to a **single agent** or **all agents** in the tenant
- Use the toggle to quickly enable/disable connectors without removing them
