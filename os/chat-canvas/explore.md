# Explore Agents

## Overview

Explore is the agent browser — the screen where you find an agent to talk to rather than configuring one you already have. It gathers every agent available to you into one searchable, filterable page and lets you start a conversation from any of them.

The page is organised into four sections, each answering a different question. **Favorites** are the agents you have starred, so the ones you use daily sit at the top. **Featured** are the agents your organization has promoted. **Custom** are the agents built inside your organization. **All Agents** is the complete list, paged with a **See more** control.

Above the sections sit a search box and a row of filters — category, subject, type, and LLM provider — that narrow every section at once. A **Create Agent** action appears for users whose role allows it.

To reach this screen, open **Explore** in the left icon rail of the chat interface.

## Target Audience

**Any User** | **Agent Builder**

## Panel Reference

#### Search
A text search across the agents available to you, matching on name and description.

#### Filters
Dropdowns that narrow the results — **Category**, **Subject**, **Type**, and **LLM Provider** — alongside a promotion filter. They combine with the search box and apply to every section on the page.

#### Favorites
The agents you have starred. This section only appears for a signed-in user, since a starred agent belongs to an account.

#### Featured
Agents promoted by your organization. This is the shelf an administrator curates for people who do not yet know what to look for.

#### Custom
Agents built inside your organization, as opposed to the defaults the platform ships with.

#### All Agents
Every agent available to you — "Explore agents and specialized learning assistants" — paged behind a **See more** control rather than loaded all at once.

#### Agent card
One card per agent, showing its avatar, name, a short description, and when it was last updated. Selecting the card opens a conversation with that agent.

#### Star toggle
The star on each card adds or removes the agent from **Favorites**. Starring requires a signed-in account.

#### Create Agent
Starts a new agent. The action is permission-gated — it is only offered to users whose role allows agent creation — and there is a matching **Create Custom Agent** card at the foot of the Custom section.

## How to Use

#### Step 1: Open Explore
In the chat interface, open **Explore** from the left icon rail. The page opens with your starred agents first, if you have any.

#### Step 2: Search or filter
Type what you are looking for, or narrow by category, subject, type, or LLM provider. The filters apply across all four sections, so a filtered page shows the matching favourites, featured, custom, and remaining agents together.

#### Step 3: Star the agents you use often
Use the star on a card to add an agent to **Favorites** so it is waiting at the top of the page next time.

#### Step 4: Start a conversation
Select an agent card to open a chat with that agent.

#### Step 5: Create one if none fits
If nothing on the page does the job and your role permits it, use **Create Agent** to build one.

## Behavior Notes

- **Favourites need an account.** Starring is stored against the signed-in user, so the Favorites section does not appear for an unauthenticated visitor.
- **Creation is permission-gated.** The **Create Agent** action only renders for roles that hold the create permission; everyone else browses without it.
- **Sections can be turned off.** Which of the four sections appear is configurable, so a deployment may show a subset — a page without a Featured shelf is a configuration, not a fault.
- **Filters narrow, they do not reorder.** The section structure stays the same under a filter; each section simply shows fewer agents.

## Building an Agent Browser Into Your Own App

The agent browser ships in the ibl.ai SDK as a self-contained component, so a product you build can offer the same discovery surface. Install the [iblai/vibe](https://github.com/iblai/vibe) skills and mount `<AgentSearch>` from `@iblai/iblai-js/web-containers/next`, passing the handlers for what a card click and a create action should do in your app; which sections render is controlled by a prop, every label is overridable, and role gating can be switched on.

For custom layouts the pieces are exported individually — the card, the search input, the filter row, the star toggle, and the empty state — along with the search and star hooks. The reusable skill for this workflow is [`iblai-vibe-agent-search`](https://github.com/iblai/vibe/tree/main/skills/iblai-vibe-agent-search), and the SDK overview is at [Vibe SDK](/developer/applications/vibe).
