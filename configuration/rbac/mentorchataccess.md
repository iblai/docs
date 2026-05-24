# Agent Chat Access

<iframe width="560" height="315" src="https://www.youtube.com/embed/jt8fL7vROcI" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Restrict a user or group to chatting with a specific agent without exposing other agents or settings on the platform. Useful for providing targeted access to internal users or external collaborators.

---

## How It Works

A pre-configured **role** and **policy** limit the user to:
- **Chat** with a specific agent
- **Read** the agent name and outputs needed for chatting
- **No access** to settings, other agents, or admin features

---

## Setup

1. Go to **Tenant Settings → Management**.
2. A role has been pre-created with the following permissions:
   - **Actions**: chat with agent, read agent metadata
   - **Data actions**: read permissions required for chatting
3. Navigate to the **Policies** tab.
4. Click **Edit** on the associated policy.
5. Verify the policy includes:
   - The correct **role**
   - The correct **platform** (tenant)
   - The specific **agent** resource
6. Under **Users**, add the user who should have chat-only access.
   - Alternatively, create a **group** and add the group to the policy.
7. Click **Save**.

---

## User Experience

When the restricted user logs in:
- The **agents list** shows only the assigned agent(s)
- The **dropdown** shows only "New Chat" (no other options)
- The **Explore** page shows no additional agents (if the agent is set to admin-only visibility)
- The user can **chat normally** with their assigned agent

---

## Adding New Users

As new users register:
1. Go to the policy and click **Edit**.
2. Add the new user to the **Users** section.
3. Save — the user can start chatting immediately.

---

## Key Takeaways

- The role and policy are **pre-configured** — you only need to add users
- Users see **only** the agent(s) assigned via their policy
- Use **groups** to manage access for multiple users at once
- This is for **chat-only** access — no settings, editing, or admin features are exposed
