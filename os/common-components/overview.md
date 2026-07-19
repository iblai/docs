# Common Components

## Overview

Many of the surfaces documented in this section are not unique to OS — they are shared components that ship from ibl.ai's common component library (`@iblai/iblai-js`) and appear consistently across ibl.ai applications.

When a component is improved in the shared library, every application that uses it picks up the change. That is why the user profile dialog, the notification center, the analytics dashboards, and the organization settings dialogs look and behave the same wherever you encounter them.

This section will collect dedicated documentation for each common component as the set grows. Today, each shared component is documented in context on the screen where you use it.

## Target Audience

**User** | **Administrator** | **Developer**

## Shared Components in OS

#### User Profile Dialog
The profile dialog — Basic Information, Education, Experience, Resume, Social, Memory, Privacy, Security, and Purchases — is a shared component. It is documented under [Profile](../profile/basic.md).

#### Notification Center
The notification inbox and alert-template management render from the shared library. They are documented under [Notifications](../notifications/inbox.md).

#### Analytics Dashboards
The Overview, Users, Topics, Transcripts, Costs, Data Reports, and Audit dashboards are shared analytics containers. They are documented under [Analytics](../analytics/overview.md).

#### Organization Settings Dialogs
The organization administration tabs — Users, Groups, Teams, Roles, Policies, Alerts, integrations, Billing, and Monetization — ship from the shared library. They are documented under [Your Organization Settings](../organization-settings/organization.md).

## Related Resources

#### Source Repositories
The OS application lives at [github.com/iblai/os](https://github.com/iblai/os). Companion tooling includes [github.com/iblai/vibe](https://github.com/iblai/vibe) (the SDK and build skills for ibl.ai applications) and [github.com/iblai/api](https://github.com/iblai/api) (agent skills and a chat MCP server for the platform API).
