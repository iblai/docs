# LLM Assignment

<iframe width="560" height="315" src="https://www.youtube.com/embed/VLKQ_tX2L9k" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Assign specific LLMs to individual users or groups so mentor editors only see the models they are authorized to use, rather than every model available to tenant admins.

---

## Prerequisites

- The user must already have **mentor editor** access (visible on the mentor's **Access** tab)
- A role with LLM-related actions must exist (or you will create one)

---

## Create an LLM Access Role

1. Go to **Tenant Settings → Management → Roles**.
2. Create a new role (e.g., "LLM Model Access").
3. Assign the actions:
   - **Read** all LLMs
   - **Select** from available LLMs

---

## Create or Edit a Policy

1. Navigate to the **Policies** tab.
2. Click **New Policy** (or edit an existing one like "LLM Model Access").
3. Configure the policy:
   - **Policy name**: descriptive label
   - **Role**: select the LLM access role created above
   - **Platform**: select your tenant
   - **LLM(s)**: choose one or more models (e.g., OpenAI GPT-4o Mini, Google, Perplexity)
4. Under **Users/Groups**, add the specific user or group.
5. Click **Save Policy**.

---

## Verify as the User

1. Log in as the assigned user.
2. Open a mentor and click the **LLMs** tab.
3. Confirm that only the assigned models appear (e.g., GPT-4o Mini, Google, Perplexity).
4. Tenant admins will still see all LLMs; restricted users see only their assigned models.

---

## Scope Options

- **Platform-wide**: the user sees the same limited LLM set across all mentors
- **Per-mentor**: restrict LLM access to specific mentors by adjusting the policy resource to a particular mentor rather than the whole platform

---

## Key Takeaways

- **Roles** define what actions can be taken (read, select LLMs)
- **Policies** bind a role to specific LLM resources and assign users or groups
- LLM restrictions apply immediately upon the user's next login
- Use **groups** to manage LLM access for multiple users at once
