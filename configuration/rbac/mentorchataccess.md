# Mentor Chat Access

<iframe width="560" height="315" src="https://www.youtube.com/embed/jt8fL7vROcI" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Restrict a user or group to chatting with a specific mentor without exposing other mentors or settings on the platform. Useful for providing targeted access to students or external users.

---

## How It Works

A pre-configured **role** and **policy** limit the user to:
- **Chat** with a specific mentor
- **Read** the mentor name and outputs needed for chatting
- **No access** to settings, other mentors, or admin features

---

## Setup

1. Go to **Tenant Settings → Management**.
2. A role has been pre-created with the following permissions:
   - **Actions**: chat with mentor, read mentor metadata
   - **Data actions**: read permissions required for chatting
3. Navigate to the **Policies** tab.
4. Click **Edit** on the associated policy.
5. Verify the policy includes:
   - The correct **role**
   - The correct **platform** (tenant)
   - The specific **mentor** resource
6. Under **Users**, add the user who should have chat-only access.
   - Alternatively, create a **group** and add the group to the policy.
7. Click **Save**.

---

## User Experience

When the restricted user logs in:
- The **mentors list** shows only the assigned mentor(s)
- The **dropdown** shows only "New Chat" (no other options)
- The **Explore** page shows no additional mentors (if the mentor is set to admin-only visibility)
- The user can **chat normally** with their assigned mentor

---

## Adding New Users

As new users register:
1. Go to the policy and click **Edit**.
2. Add the new user to the **Users** section.
3. Save — the user can start chatting immediately.

---

## Key Takeaways

- The role and policy are **pre-configured** — you only need to add users
- Users see **only** the mentor(s) assigned via their policy
- Use **groups** to manage access for multiple users at once
- This is for **chat-only** access — no settings, editing, or admin features are exposed
