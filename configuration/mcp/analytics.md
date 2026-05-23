# Analytics

<iframe width="560" height="315" src="https://www.youtube.com/embed/Q88dvtE3wVQ" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

---

## Purpose

Connect an Analytics MCP server to an agent so it can query, analyze, and visualize platform data—including user activity, learner metrics, content usage, financial analytics, and message/session data—directly in chat.

---

## Overview

An Analytics MCP–powered agent acts as an Analytics Assistant, pulling real data from your platform via MCP. It can answer questions, generate summaries, and return visualizations (e.g., graphs) based on up-to-date analytics.

---

## Prerequisites

- An agent configured with an Analytics Assistant system prompt.  
- MCP Tool enabled for the agent.  
- An API key for the Analytics MCP server.

---

## Setup Steps

#### 1) Configure the Agent

- Open the agent you want to use for analytics.  
- Set the system prompt to an Analytics Assistant that can pull:
  - User and platform analytics  
  - Learner metrics  
  - Content analytics  
  - Financial analytics  
  - Message and session data  

#### 2) Enable the MCP Tool

- Go to the agent’s **Tools** tab.  
- Toggle **MCP** to **On**.

#### 3) Add the Analytics MCP Connector

- Open the **MCP** tab.  
- Add (or edit) a connector with:
  - **Connector name**
  - **Server location** (Analytics MCP URL)
  - **Description**  
      - Similar to a system prompt; describe what data can be pulled and how responses should be formed (e.g., user/platform analytics, learner metrics, content usage, financials).
  - **Transport:** Streamable HTTP
  - **Authentication:** API Key
      - **Token type:** API Key
      - **Token value:** paste your API key (hidden after save)
- Save the connector.

---

## Using Analytics in Chat

#### Example Queries

**Prompt example:**

> “Show me a graph of last logged-in users over the past seven days.”

- The agent retrieves data across multiple sources and returns a visualization and counts.

**Prompt example:**

> “What agent has the highest usage over the past seven days?”

- The agent identifies the top-used agent and can include associated metrics (e.g., LLM cost).

**Note:** Some queries may take slightly longer due to multi-step data retrieval, but responses remain fast.

---

## Result

With Analytics MCP connected, agents can answer analytics questions, generate graphs, and summarize usage and costs—all grounded in live platform data and delivered directly in conversation.
