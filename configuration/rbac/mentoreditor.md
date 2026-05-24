# Agent Editor

<iframe width="560" height="315" src="https://www.youtube.com/embed/WpJBhNB-2xs" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Grant edit access to a specific agent without elevating a user’s tenant-wide permissions. This lets administrators or tenant admins collaborate on an agent while keeping access tightly scoped.

---

## Overview

- Users without editor rights cannot edit any agents by default.  
- Tenant Admins can edit any agent and assign editor access.  
- Administrators can grant editor access only to agents they own.  
- Editor access is agent-specific—not tenant-wide.

---

## Granting Editor Access (Owner/Admin)

### 1. Open the Agent
- Go to the agent you want to share (e.g., **Socratic Agent**).

### 2. Open Access Settings
- In **Agent Settings**, select the **Access** tab.

### 3. Assign the Editor Role
- Use the existing **Editor** role (or create one if needed).  
- Add the user (by email/username) to the **Editor** role.  
- Click **Save**.

This grants edit rights **only for this agent**.

---

## What Editors Can Do

Once assigned, the user can:

- Edit settings  
- Change the LLM  
- Update system prompts  
- Add/remove data sets  
- Enable/disable tools  

They cannot:

- Access tenant settings  
- Edit other agents they weren’t granted access to  

---

## Verifying Access (Editor’s View)

- The user remains a standard user (or non-admin) in profile.  
- The shared agent shows **Edit** options in the dropdown.  
- Other agents still show **Chat only** (no edit access).

---

## Notes & Best Practices

- Use editor access for collaboration without over-privileging users.  
- Prefer agent-level editors over tenant roles to minimize risk.  
- Review editor assignments periodically from the agent’s **Access** tab.

---

## Result

You can safely collaborate by giving users precise, agent-level edit access—no broader permissions required.


