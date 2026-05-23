# Team Manager Assignment

<iframe width="560" height="315" src="https://www.youtube.com/embed/3l7oPOtcogI" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Restrict visibility so external partners (e.g., companies purchasing courses) can only see analytics for their own learners, not data from other organizations or teams.

---

## Overview

This setup uses roles, policies, and groups to grant scoped analytics access. Team managers can view reports for the learners they oversee—nothing else.

---

## Setup Steps

### 1) Create or Use a Team-Scoped Role

- Go to **Tenant Settings → Management → Roles**.
- Use (or create) a role like **Company Analytics Viewer** with actions such as:
  - Read analytics and reports
  - Read specific user groups
- This role should **not** include content editing or invitation permissions.

### 2) Bind the Role to a Specific Group (Policy)

- Navigate to **Management → Policies**.
- Create (or edit) a policy for the company/team (e.g., **Company A**).
- Assign the **Company Analytics Viewer** role.
- Set the **Resource** to:
  - The platform
  - A specific user group (the team this company should see)
- Click **Save**.

This resource binding is what limits visibility to only that group’s learners.

### 3) Assign Users to the Policy

- In the same policy, add:
  - Individual users, or
  - A group that represents the company’s managers
- Click **Save**.

Anyone added here gains analytics access **only** for the selected group.

---

## What Team Managers See (User View)

- Access to **Analytics dashboards**.
- Ability to **filter reports** by the groups they oversee.
- Reports and dashboards show **only their assigned learners**.
- **No visibility** into other companies’ users or data.
- Current dashboards focus on **agent analytics**; course dashboards will be added later.

---

## Example Outcome

A manager assigned to **Group B**:
- Sees analytics filtered to **Group B only**.
- Cannot view data for **Group A** or any other teams.

---

## Result

By pairing a **team-scoped role** with a **group-bound policy**, you ensure external managers have the insights they need—without exposing any other learner data.
