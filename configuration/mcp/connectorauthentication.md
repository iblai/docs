# MCP Connector Authentication

<iframe width="560" height="315" src="https://www.youtube.com/embed/fcFsqz7PndI" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Overview

Configure MCP connectors with different authentication methods and scopes to control how users authenticate with external services like Notion, Google, and other OAuth-based providers.

---

## Adding a Custom MCP Connector

1. Go to the **MCP** tab on a mentor.
2. Click **Add Custom MCP Connector**.
3. Provide the **connector URL**.
4. Select the **connector scope**:
   - **This mentor only**: the connector is available only for this mentor
   - **All mentors**: the connector is available for every mentor in the tenant

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
| **Tenant** | Authenticate once — the authentication is shared across all tenant members and all mentors |
| **Mentor** | Authenticate once — the authentication applies to this mentor only but is shared across all users |
| **User** | Each user must authenticate individually when they first interact with the mentor |

---

## User-Based Authentication Flow

1. Admin adds the OAuth MCP connector with **User** scope.
2. Admin authenticates first (completes the OAuth flow).
3. The connector is enabled on the mentor.
4. When a different user (e.g., a student) chats with the mentor:
   - An authentication prompt appears: "Authentication required for [service]. Please complete the login in the open window."
   - The user completes the OAuth flow in a popup window.
   - Upon success, the chat continues with the connector active.

---

## Enable/Disable Connectors

- After adding a connector, use the **toggle** to enable or disable it.
- **Enabled**: the mentor can use the MCP connector in responses.
- **Disabled**: the mentor cannot access the connector.

---

## Key Takeaways

- **Tenant** scope: authenticate once, everyone benefits
- **Mentor** scope: authenticate once per mentor, shared across users
- **User** scope: every user authenticates individually (most secure)
- Connectors can be scoped to a **single mentor** or **all mentors** in the tenant
- Use the toggle to quickly enable/disable connectors without removing them
