# Global Roles

## Purpose

Assign platform-wide capabilities to individual users without building custom roles and policies. Global roles are set by tenant admins under **Management → Users**.

---

## Available Roles

### Analytics Viewer

Grants access to the AI Analytics dashboard on skills.iblai.app. The user can see aggregate analytics for teams they have analytics access to through [team sharing](teamsharing.md). This does **not** grant access to analytics for specific agents.

### Billing Manager

Can view and manage platform credits (reloading credits, etc.).

### Can Sell Items

Can configure items that can be sold on the platform.

### Create Teams

Can create teams of individuals under **Management → Teams**.

### Enrollment Manager

Can invite others to the platform. See [Enrollment Manager Assignment](enrollmentmanagerassignment.md) for detailed setup.

### List Teams

Can list teams the current user has access to. Typically used in conjunction with other permissions that require selecting teams, such as viewing aggregate analytics.

### List Users

Can list users on the platform. Typically used in conjunction with other permissions that require selecting users, such as when creating teams.

### Agent Chat

Can chat with **any** agent in the tenant. See [Agent Chat](mentorchataccess.md) for scoped, per-agent chat access.

### Agent Editor

Can chat with any agent and view/edit all settings and information about **all** agents in the tenant. See [Agent Editor](mentoreditor.md) for scoped, per-agent editor access.

### Agent Viewer

Can chat with any agent and view all settings and information about all agents in the tenant (read-only).

### Notification Manager

Can access alerts management and send notifications to users/teams they have access to through [team sharing](teamsharing.md).

---

## How to Assign a Global Role

1. Go to **Tenant Settings → Management**.
2. Open the **Users** tab.
3. Select the user you want to modify.
4. Check/Uncheck the desired roles.

--- 

Assigning some roles through team sharing will automatically enable specific global roles for that user/group such as analytics viewer, notification manager, list team, etc as necessary.
