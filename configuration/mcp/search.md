# Search

<iframe width="560" height="315" src="https://www.youtube.com/embed/HbKNTemQeLU" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

## Purpose

Connect a Search MCP to an agent so it can search your platform for courses, programs, and agents—and return grounded, recommendation-ready results without hallucinations.

---

## Overview

The Search MCP uses an MCP server (in this demo, ibl.ai’s own search MCP) to power an agent that acts as a search assistant. It can:

- Search the course catalog (courses/programs).
- Find and recommend agents.
- Ask follow-up filters (subject, level, format, language).
- Enforce guardrails via the system prompt.

---

## Prerequisites

- An agent with a Search Assistant system prompt.
- MCP Tool enabled for the agent.
- An API key for the Search MCP server.

---

## Setup Steps

#### 1) Prepare the Agent

- Open the agent you want to use for search.
- Set the system prompt to a search-focused assistant (catalog + agent search, recommendations, guardrails).

#### 2) Enable the MCP Tool

- Go to the agent’s **Tools** tab.
- Ensure **MCP** is enabled.

#### 3) Add the Search MCP Connector

- Open the **MCP** tab.
- Add (or edit) a connector with:
  - **Connector name**
  - **Server location** (Search MCP server URL)
  - **Description**  
     - Acts like instructions for the connection (what the agent can pull and how to respond).
  - **Transport:** Streamable HTTP
  - **Authentication:** API Key
     - **Token type:** API Key
     - **Token value:** paste your token
- Save the connector.

---

## Using Search in Chat

#### Search Courses

**Prompt example:**

> “What courses are available on the platform?”

**Result:**

- Returns a subset of courses from the tenant (even if the catalog is large).
- Asks if you want to filter by subject, level, format, or language.

#### Search Agents

**Prompt example:**

> “What agents can help me become a better student?”

**Result:**

- Lists relevant agents (e.g., study tips, quizzes, Socratic support).
- Offers to filter further or show more results (based on system-prompt limits).

---

## Result

With Search MCP connected, agents can reliably discover courses and agents across your platform, guide users with filters and recommendations, and return accurate, grounded results from your own data.
