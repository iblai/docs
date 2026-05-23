# Moodle LTI 1.3 Deep Linking

<iframe width="560" height="315" src="https://www.youtube.com/embed/oOPGSEaiE4U" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Prerequisites

- You must be a Moodle **site administrator**.
- Obtain all LTI parameters from the Agentic OS team (tool URL, key set URL, login URL, etc.).

---

## Register the Tool

1. Go to **Site Administration → Plugins → Activity Modules → Manage Tools**.
2. Click **Configure a Tool Manually**.
3. Fill in the fields:
   - **Tool name**: Agent AI
   - **Tool URL**: provided by Agentic OS
   - **Key set URL**: provided by Agentic OS
   - **Login URL**: provided by Agentic OS
   - **LTI version**: **1.3**
4. Under **Tool Configuration Usage**, select **Show as preconfigured tool and adding an external tool**.
5. Set **Default Launch Container** to **Existing Window** (or choose Embed, Embed Without Blocks, or New Window).
6. Check **Deep Linking** to enable content selection.
7. Under **Privacy**, enable sharing the user's **name** and **email** for reporting purposes.
8. Save the configuration.

---

## Add the Agent to a Course

1. Navigate to the course where you want to add the agent.
2. Click **Edit Mode** to enable editing.
3. Click the **+ Activity or Resource** button.
4. Select **Agent AI** from the list (under Activities or Starred).
5. Click **Select Content** — a window opens showing all available agents.
6. Choose the agent you want to integrate (e.g., Career Path Agent).
7. Click **Select Content**.
8. Optionally add a description or configure the module settings.
9. Click **Save and Display**.

---

## Result

- The agent loads directly in the course page.
- Students can chat with the agent immediately.
- The embedded size can be adjusted from the integration settings.

---

## Key Takeaways

- **LTI version must be 1.3** — verify this in the tool configuration
- **Deep linking** allows instructors to select specific agents per course module
- Enable **name and email sharing** for accurate learner analytics
- The agent can be added to multiple courses with different agent selections via deep linking
