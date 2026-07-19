# Agent Settings: Skills

![Skills panel with a New Skill button and two skill cards, Web Research v1.0.0 toggled on and Image Creation v1.0.0 toggled off, each with an overflow actions menu](/images/docs/os/agent-settings/agent_settings_skills.webp)

## Overview

The Skills panel assigns reusable skills to an agent. The panel header reads: "Manage agent skill assignments for your mentor" (the UI's exact wording), and the helper text explains: "Skills are reusable capabilities you build once and assign to any agent. Create a skill, then toggle it on to give this agent that ability."

Skills live at the organization level: you define a skill once (name, slug, description, version, and instruction text) and can then assign it to any number of agents. Each skill appears as a card with its name, a version badge (for example, v1.0.0), an assignment toggle, and an overflow ("...") menu for managing the skill itself.

To reach this screen, open the **Edit Agent** modal, stay on the **Configurations** tab group, and select **Skills** in the left sidebar.

## Target Audience

**Administrator** | **Agent Builder**

## Panel Reference

#### New Skill
Opens the skill-creation dialog. **Name** and **Slug** are required; you can also provide a description, a version (defaults to 1.0.0), and the skill's instruction text — the behavior the skill adds when assigned to an agent. Newly created skills are enabled on creation, and a "Skill created" confirmation appears.

#### Skill cards
Each card lists a skill available in the organization, with its name and version (the screenshot shows **Web Research** v1.0.0 and **Image Creation** v1.0.0). The card's toggle controls the skill's assignment to this agent: on means the agent has the skill (Web Research in the screenshot), off means it does not (Image Creation).

#### Assignment toggle
Toggling on creates a skill assignment for this agent; toggling off removes it. Because skills are shared, toggling here only affects this agent's assignment — the skill itself remains available to other agents.

#### Overflow ("...") menu
Actions for the underlying skill, including editing its definition (name, slug, description, version, instruction) and deleting it. Editing or deleting a skill affects it organization-wide, not just for this agent; deleting shows a "Skill deleted" confirmation.

## How to Use

#### Step 1: Open the Skills panel
Open the **Edit Agent** modal, keep **Configurations** selected, and click **Skills** in the sidebar.

#### Step 2: Create a skill (if needed)
Click **New Skill**, fill in at least the required **Name** and **Slug**, add a description and instruction text describing the capability, and save. The skill becomes available to every agent in the organization.

#### Step 3: Assign skills to this agent
Toggle on each skill this agent should have — for example, Web Research for an agent that should research on the web. The assignment takes effect for this agent's conversations.

#### Step 4: Maintain skills
Use the "..." menu to edit a skill's definition or delete it. Remember these actions apply organization-wide; to remove a capability from just this agent, use the toggle instead.
