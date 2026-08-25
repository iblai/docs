# Agent Settings: Memory

![Memory panel with an enable toggle, user and date filters, category tabs such as Knowledge Gaps and Preferences, an Add Memory button, and saved memory cards with timestamps and user emails](/images/docs/os/agent-settings/agent_settings_memory.webp)

## Overview

The Memory panel reviews and manages what an agent remembers about individual users across conversations. The panel header reads: "Configure memory settings for your agent," and the helper text explains: "Let this agent remember useful details from past conversations — like a returning colleague — so people don't have to repeat themselves. When on, review and manage what it remembers below," with a master toggle to turn memory on or off for the agent.

Each saved memory is a card showing when it was captured (for example, "less than a minute ago"), which user it belongs to (their email), and the remembered fact (for example, "Prefers project-based learning over lecture-based content."). Memories are organized into categories and can be filtered, added, edited, and deleted by administrators.

The panel covers two different kinds of memory. Most cards are **about a user** — what the agent remembers about one person. The agent also keeps memory **of its own**, shared across everyone who talks to it: see [Agent-owned memory](#agent-owned-memory) below.

To reach this screen, open the **Edit Agent** modal, switch to the **Runtime** tab group (its sidebar lists Tasks, Memory, History, Audit, and Analytics), and select **Memory**.

## Target Audience

**Administrator** | **Agent Builder**

## Panel Reference

#### Memory toggle
The switch in the helper banner enables or disables memory for this agent. When on, the agent stores and references useful details from conversations; when off, it does not remember users between sessions. (Conversations in private mode never store memory regardless of this setting.)

#### Search for User
A searchable dropdown listing the users who have saved memories (by email), plus an **All Users** option. Selecting a user filters the memory list to just that person.

#### Pick a Date Range
Filters the memory list to entries captured within a chosen date range.

#### Category tabs
Filter tabs across the top of the list: **All** plus the organization's memory categories — the screenshot shows **Knowledge Gaps**, **Learning Goals**, **Personal Context**, and **Preferences**. Up to six categories are shown inline; additional ones collapse into a **More** menu. Categories are admin-defined, so your set may differ.

#### Categories
Opens the **Manage Categories** dialog, where you can add a new category by name, rename existing ones, and delete them (with confirmation). Categories structure how memories are classified and filtered.

#### Add Memory
Opens the **Add Memory** dialog: pick a **Category**, type the **Memory** content (10-character minimum), and click **Save**. This lets an administrator seed or correct what the agent knows about users without waiting for it to be learned in conversation.

#### Memory cards
Each card shows the memory's relative timestamp, the owning user's email, and the memory text. The "..." actions menu on a card offers **Edit** (change the content or category) and **Delete** (with confirmation). A **Delete All** action is available to bulk-delete the memories in the current category filter.

## Agent-owned memory

An agent is usually used by more than one person — a whole office, a whole course, a whole support team. **Agent-owned memory is what the agent knows that is not about any particular one of them**, and it is available to every user of that agent rather than to a single person.

The practical difference:

| | User memory | Agent-owned memory |
|---|---|---|
| Subject | One person | The agent itself |
| Visible to | That person's conversations | Everyone who uses the agent |
| Example | "Prefers project-based learning." | "Transfer credits from the community college are evaluated against the 2026 catalog, not the year of enrolment." |

**Use it for what the whole team should not have to re-learn** — a correction one colleague made that everyone should inherit, a recurring edge case and how it was settled, an operating convention the office agreed on, a local fact the agent had to be told once. The test: *if the next person to open a conversation should benefit from it, it is agent-owned; if it would be wrong to surface to a different person, it belongs to that person's memory instead.*

Because these entries are visible to every user of the agent, **keep personal data out of them** — anything identifying an individual belongs in that individual's memory. Agent-owned entries are reviewable, editable, and deletable by administrators with edit access to the agent, and changes appear in the agent's [Audit](audit.md) trail. They do not propagate to other agents; an agent that should share knowledge with another is given the same skills or datasets instead.

## How to Use

#### Step 1: Open the Memory panel
Open the **Edit Agent** modal, click the **Runtime** tab group, then select **Memory**.

#### Step 2: Turn memory on
Enable the toggle in the banner so the agent remembers useful details from past conversations. Pair this with the **Remember past conversations** capability so memory is active in chat.

#### Step 3: Review what the agent remembers
Filter by user with **Search for User**, narrow by date with **Pick a Date Range**, and switch category tabs (Knowledge Gaps, Learning Goals, Personal Context, Preferences, and so on) to audit stored memories.

#### Step 4: Curate memories
Use **Add Memory** to insert facts the agent should know, the card's **Edit** action to fix inaccurate ones, and **Delete** (or **Delete All**) to remove anything that should not be retained.

#### Step 5: Maintain categories
Click **Categories** to add, rename, or remove memory categories as your classification needs evolve.
