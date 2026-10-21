# Memory System

The Memory System lets AI agents carry knowledge across conversations — both what they learn about an individual user, and what an administrator wants the agent itself to know for everyone who uses it. It stores user preferences, learning progress, knowledge gaps, personal context, and hand-curated agent knowledge, making interactions more personalized, more consistent, and less repetitive.

---

## Overview

The memory system supports three scopes of memory. The first two are **about a user**; the third is **about the agent**.

| Memory Type | Scope | Subject | Written by | Description |
|-------------|-------|---------|------------|-------------|
| **Global User Memories** | All agents | The user | Auto-extracted from chat, or by hand | Facts about the user that apply everywhere (name, profession, preferences) |
| **Agent-Specific Memories** | Single agent | The user | Auto-extracted per category, or by hand | Context specific to *this user's* interactions with a particular agent, filed by category |
| **Agent Knowledge** (Shared Memory) | Single agent | The agent | **By hand only** | Facts, rules and background every person who chats with the agent receives, injected as an `## Agent Knowledge` block |

The distinction that matters: **Agent-Specific Memories** are still keyed to a person — they are what the agent remembers *about you*. **Agent Knowledge** belongs to the agent itself, has no user and no category, is never extracted automatically, and is delivered to everyone it serves. In the OS it is the **Shared Memory** sub-tab of the agent's Memory panel; see [Agent Knowledge API](#agent-knowledge-api-shared-memory).

**Default Memory Categories (Agent-Specific):**

| Category | Slug | Description |
|----------|------|-------------|
| Knowledge Gaps | `knowledge_gaps` | Topics or concepts the user struggles with |
| Learning Goals | `learning_goals` | Goals and objectives the user wants to achieve |
| Preferences | `preferences` | Learning style, pace, or content preferences |
| Progress Milestones | `progress_milestones` | Achievements and completed learning milestones |
| Personal Context | `personal_context` | Relevant personal information shared by the user |

---

## Architecture

### System Overview

The memory system uses PGVector for semantic search and consists of four main components that work together to extract, store, and retrieve memories.

The data flows through two paths:

1. **Extraction path (write):** A Chat Session feeds into a background Celery Task containing the Extraction Service, which processes conversations and writes to the Memory Store (PGVector) — storing global memories, agent memories, and their embeddings.
2. **Retrieval path (read):** When generating a new chat response, the Context Service queries the Memory Store via Semantic Search (cosine distance), retrieves relevant memories, and injects them into the AI's context to produce a personalized response.

### Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **MemoryExtractionService** | `services/memory_extraction.py` | Extracts memories from conversations using LLM |
| **MemoryStore** | `services/memory_store.py` | Handles storage, deduplication, and retrieval |
| **MemoryContextService** | `services/memory_context.py` | Retrieves and formats memories for chat injection |
| **Celery Tasks** | `tasks.py` | Background processing for memory extraction |

---

## Memory Extraction Flow

When a user sends a message, the system automatically extracts relevant memories in the background.

```
┌──────────────────┐
│ User sends       │
│ message          │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ AI generates     │
│ response         │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Celery task      │
│ triggered        │
│ (background)     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌─────────────────┐
│ Check settings   │────▶│ SKIP            │
│ - Tenant enabled?│ No  │ (not enabled)   │
│ - Agent enabled?│     └─────────────────┘
│ - User enabled?  │
└────────┬─────────┘
         │ Yes
         ▼
┌──────────────────┐
│ Get existing     │
│ memories summary │
│ (for context)    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ SINGLE LLM CALL  │
│                  │
│ Input:           │
│ - User message   │
│ - AI response    │
│ - Categories     │
│ - Existing mem   │
│                  │
│ Output:          │
│ - has_memories   │
│ - global_memories│
│ - mentor_memories│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌─────────────────┐
│ has_memories?    │────▶│ SKIP            │
│                  │ No  │ (nothing to     │
│                  │     │  extract)       │
└────────┬─────────┘     └─────────────────┘
         │ Yes
         ▼
┌──────────────────┐
│ Deduplication    │
│                  │
│ 1. Hash check    │───▶ Skip if exact match
│ 2. Semantic check│───▶ Skip if similar (cosine < 0.15)
└────────┬─────────┘
         │ Unique
         ▼
┌──────────────────┐
│ Generate         │
│ embedding        │
│ (1536 dimensions)│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Store in         │
│ PostgreSQL       │
│ with PGVector    │
└──────────────────┘
```

### Extraction Details

The extraction service uses a **single LLM call** to both decide if extraction is needed AND extract memories. This optimization reduces cost and latency.

**LLM Input:**
```
Categories:
- knowledge_gaps: Topics or concepts the user struggles with
- learning_goals: Goals and objectives the user wants to achieve
...

Existing memories (avoid duplicates):
Global:
  - The user is a software engineer
Agent-specific:
  - [learning_goals] The user wants to learn Python

Latest Exchange:
User: I'm having trouble understanding recursion
Assistant: Let me explain recursion step by step...
```

**LLM Output:**
```json
{
  "has_memories": true,
  "global_memories": [],
  "mentor_memories": {
    "knowledge_gaps": ["The user struggles with understanding recursion"]
  }
}
```

### Deduplication Strategy

The system uses a **3-layer deduplication** approach to prevent duplicate memories:

| Layer | Method | Purpose |
|-------|--------|---------|
| 1 | SHA-256 Hash | Fast check for exact duplicates |
| 2 | Semantic Similarity | PGVector cosine distance (threshold: 0.15) for near-duplicates |
| 3 | LLM Context | Existing memories shown to LLM to inform extraction |

---

## Memory Injection Flow

When a user starts a new conversation, relevant memories are retrieved and injected into the AI's context.

```
┌──────────────────┐
│ User sends       │
│ new message      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌─────────────────┐
│ Check agent     │────▶│ NO INJECTION    │
│ memory enabled?  │ No  │                 │
└────────┬─────────┘     └─────────────────┘
         │ Yes
         ▼
┌──────────────────┐     ┌─────────────────┐
│ Check user       │────▶│ NO INJECTION    │
│ use_memory       │ No  │                 │
│ enabled?         │     └─────────────────┘
└────────┬─────────┘
         │ Yes
         ▼
┌──────────────────┐
│ Generate query   │
│ embedding from   │
│ user message     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Semantic search  │
│                  │
│ - Top 5 global   │
│ - Top 5 agent   │
│                  │
│ (ordered by      │
│  cosine distance)│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Format as        │
│ markdown context │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Inject into      │
│ system prompt    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ AI generates     │
│ personalized     │
│ response         │
└──────────────────┘
```

### Injected Context Format

The AI receives memories formatted as:

```markdown
## Agent Knowledge
- Our refund window is 30 days
- Always cite the policy handbook

## User Information
- The user is a software engineer with 5 years of experience
- The user prefers visual explanations with diagrams

## Relevant Context from Previous Conversations
- [Knowledge Gaps] The user struggled with understanding recursion
- [Learning Goals] The user wants to master system design patterns
- [Preferences] The user prefers Python code examples over pseudocode
```

`## Agent Knowledge` is the agent's shared memory and is the same for every user; the two user sections are retrieved per person by semantic search (roughly the top five global and top five agent-specific memories). Because the knowledge block reaches every user's prompt, it is written only by hand — there is no code path that extracts it from a conversation.

---

## Configuration Hierarchy

Memory features require enablement at three levels:

```
┌─────────────────────────────────────────────────────────────┐
│                      TENANT LEVEL                           │
│                                                             │
│  Tenant Profile → Memory Tab                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Memory Configuration: [ENABLED/DISABLED]            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          │ If disabled, stops here          │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   MENTOR LEVEL                       │   │
│  │                                                      │   │
│  │  Agent Settings → Memory Tab                        │   │
│  │  ┌────────────────────────────────────────────┐     │   │
│  │  │  Enable Memory: [ON/OFF]                    │     │   │
│  │  │  Memory Categories: [Configure...]          │     │   │
│  │  └────────────────────────────────────────────┘     │   │
│  │                       │                              │   │
│  │                       │ If disabled, stops here      │   │
│  │                       ▼                              │   │
│  │  ┌────────────────────────────────────────────┐     │   │
│  │  │                USER LEVEL                   │     │   │
│  │  │                                             │     │   │
│  │  │  User Profile → Memory Settings             │     │   │
│  │  │  ┌───────────────────────────────────┐     │     │   │
│  │  │  │ Auto-capture memories: [ON/OFF]   │     │     │   │
│  │  │  │ Use memories in responses: [ON/OFF]│     │     │   │
│  │  │  └───────────────────────────────────┘     │     │   │
│  │  └────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## SPA Configuration Guide

### Enabling Memory for a Tenant

To enable the memory feature for your entire platform:

1. Navigate to **Tenant Profile** page
2. Select the **Memory** tab
3. Toggle the memory configuration switch to **enabled**

> **Note:** This is the master switch. Memory features will not work for any agent or user until this is enabled.

---

### Enabling Memory for an Agent

Each agent can have memory features enabled or disabled individually:

1. Navigate to **Agent Settings** for the specific agent
2. Select the **Memory** tab
3. Toggle memory to **enabled**

Once enabled, the agent will:
- Automatically extract memories from conversations
- Use stored memories to personalize responses

**Managing Agent Memory Categories:**

From the Agent Settings → Memory tab, administrators can:
- View default memory categories
- Create custom categories
- Edit category names and descriptions
- Deactivate categories (memories in that category will no longer be extracted)

**Viewing User Memories for an Agent:**

The Agent Settings → Memory tab also displays all memories stored for users interacting with that agent, organized by category.

---

### User Memory Settings

Users control their own memory preferences from their profile:

1. Navigate to **User Profile** page
2. Locate the memory settings section
3. Configure the following options:

| Setting | Description |
|---------|-------------|
| **Auto-capture memories** | When enabled, the system automatically extracts and saves memories from conversations |
| **Use memories in responses** | When enabled, stored memories are used to personalize AI responses |

> **Note:** Users can disable memory features entirely for privacy, even if the tenant and agent have memory enabled.

---

### Managing Global User Memories

Global memories are facts about the user that apply across all agents (e.g., "The user is a software engineer").

**Location:** User Profile page → Global Memories section

**User Actions:**

| Action | Description |
|--------|-------------|
| **View memories** | See all automatically captured and manually added global memories |
| **Add memory** | Manually add a new global memory |
| **Delete memory** | Remove a memory that is no longer relevant or accurate |

---

### Managing Agent-Specific Memories

Agent memories are specific to a user's interactions with a particular agent.

**Location:** Agent Settings page → Memory tab → User Memories section

**User/Admin Actions:**

| Action | Description |
|--------|-------------|
| **View memories** | See all memories organized by category |
| **Filter by category** | View memories for a specific category (e.g., Knowledge Gaps) |
| **Add memory** | Manually add a memory to a specific category |
| **Edit memory** | Update the content of an existing memory |
| **Delete memory** | Remove a memory |

---

## API Reference

Base URL: `/api/ai-agent/`

### Global Memories API

**Endpoints:** `/orgs/{org}/users/{user_id}/global-memories/`

#### List Global Memories

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/global-memories/" \
  -H "Authorization: Token <token>"
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `session_id` | string | Filter memories by source session ID |
| `content` | string | Case-insensitive search within memory content |
| `start_date` | ISO 8601 date | Filter memories created on or after this date |
| `end_date` | ISO 8601 date | Filter memories created on or before this date |

**Response:**
```json
{
  "count": 2,
  "results": [
    {
      "id": 1,
      "content": "The user is a software engineer with 5 years of experience",
      "is_auto_generated": true,
      "created_at": "2024-01-15T10:30:00Z"
    },
    {
      "id": 2,
      "content": "The user prefers detailed code examples",
      "is_auto_generated": true,
      "created_at": "2024-01-14T09:15:00Z"
    }
  ]
}
```

**Filtering Examples:**

**Filter by session:**
```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/global-memories/?session_id=abc-123-def" \
  -H "Authorization: Token <token>"
```

**Search by content:**
```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/global-memories/?content=software%20engineer" \
  -H "Authorization: Token <token>"
```

**Filter by date range:**
```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/global-memories/?start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Token <token>"
```

**Combined filters:**
```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/global-memories/?content=python&start_date=2024-01-01" \
  -H "Authorization: Token <token>"
```

#### Create Global Memory

```bash
curl -X POST \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/global-memories/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "The user is preparing for a job interview"
  }'
```

**Response:**
```json
{
  "id": 3,
  "content": "The user is preparing for a job interview",
  "is_auto_generated": false,
  "created_at": "2024-01-16T14:00:00Z"
}
```

#### Delete Global Memory

```bash
curl -X DELETE \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/global-memories/3/" \
  -H "Authorization: Token <token>"
```

**Response:** `204 No Content`

---

### Agent Memories API

**Endpoints:** `/orgs/{org}/users/{user_id}/agents/{mentor_id}/agent-memories/`

#### List All Agent Memories (All Agents)

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/agent-memories/" \
  -H "Authorization: Token <token>"
```

**Response:**
```json
{
  "count": 3,
  "results": [
    {
      "id": 1,
      "agent": "python-tutor",
      "category": {
        "id": 1,
        "name": "Knowledge Gaps",
        "slug": "knowledge_gaps"
      },
      "content": "The user struggled with understanding recursion",
      "is_auto_generated": true,
      "created_at": "2024-01-15T11:00:00Z"
    },
    {
      "id": 2,
      "agent": "python-tutor",
      "category": {
        "id": 2,
        "name": "Learning Goals",
        "slug": "learning_goals"
      },
      "content": "The user wants to master data structures",
      "is_auto_generated": true,
      "created_at": "2024-01-14T16:30:00Z"
    }
  ]
}
```

#### List Memories for Specific Agent

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/agents/python-tutor/agent-memories/" \
  -H "Authorization: Token <token>"
```

#### Filter by Category

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/agents/python-tutor/agent-memories/?category=knowledge_gaps" \
  -H "Authorization: Token <token>"
```

#### Create Agent Memory

```bash
curl -X POST \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/agents/python-tutor/agent-memories/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "The user completed the Python basics course",
    "category_slug": "progress_milestones"
  }'
```

**Response:**
```json
{
  "id": 4,
  "agent": "python-tutor",
  "category": {
    "id": 4,
    "name": "Progress Milestones",
    "slug": "progress_milestones"
  },
  "content": "The user completed the Python basics course",
  "is_auto_generated": false,
  "created_at": "2024-01-16T15:00:00Z"
}
```

#### Update Agent Memory

```bash
curl -X PATCH \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/agents/python-tutor/agent-memories/4/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "The user completed Python basics and intermediate courses"
  }'
```

#### Delete Agent Memory

```bash
curl -X DELETE \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/agents/python-tutor/agent-memories/4/" \
  -H "Authorization: Token <token>"
```

**Response:** `204 No Content`

---

### User Memory Settings API

**Endpoints:** `/orgs/{org}/users/{user_id}/memsearch-settings/`

#### Get User Settings

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/memsearch-settings/" \
  -H "Authorization: Token <token>"
```

**Response:**
```json
{
  "auto_capture_enabled": true,
  "use_memory_in_responses": true,
  "updated_at": "2024-01-15T10:00:00Z"
}
```

#### Update User Settings

```bash
curl -X PUT \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/users/user123/memsearch-settings/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "auto_capture_enabled": true,
    "use_memory_in_responses": false
  }'
```

**Response:**
```json
{
  "auto_capture_enabled": true,
  "use_memory_in_responses": false,
  "updated_at": "2024-01-16T16:00:00Z"
}
```

---

### Memory Categories API

**Endpoints:** `/orgs/{org}/agents/{mentor_id}/memory-categories/`

#### List Memory Categories

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/agents/python-tutor/memory-categories/" \
  -H "Authorization: Token <token>"
```

**Response:**
```json
{
  "count": 5,
  "results": [
    {
      "id": 1,
      "name": "Knowledge Gaps",
      "slug": "knowledge_gaps",
      "description": "Topics or concepts the user struggles with",
      "is_active": true
    },
    {
      "id": 2,
      "name": "Learning Goals",
      "slug": "learning_goals",
      "description": "Goals and objectives the user wants to achieve",
      "is_active": true
    }
  ]
}
```

#### Create Custom Category

```bash
curl -X POST \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/agents/python-tutor/memory-categories/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Project Ideas",
    "slug": "project_ideas",
    "description": "Project ideas the user has expressed interest in"
  }'
```

#### Update Category

```bash
curl -X PATCH \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/agents/python-tutor/memory-categories/6/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Project ideas and coding challenges the user wants to try"
  }'
```

#### Deactivate Category

```bash
curl -X DELETE \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/agents/python-tutor/memory-categories/6/" \
  -H "Authorization: Token <token>"
```

> **Note:** Deleting a category deactivates it (`is_active: false`). Existing memories in that category are preserved but no new memories will be extracted for it.

---

### Agent Knowledge API (Shared Memory)

**Endpoints:** `/orgs/{org}/agents/{agent}/agent-memories/`

Agent knowledge is scoped to the agent, so the path has **no** `users/{user_id}` segment. Do not confuse it with the user-scoped `/orgs/{org}/users/{user_id}/agents/{agent}/agent-memories/` above, which lists one user's per-agent memories. The two share the `agent-memories` name and nothing else.

#### List Agent Knowledge

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/agents/python-tutor/agent-memories/?page=1&page_size=20" \
  -H "Authorization: Token <token>"
```

Returns a paged list (`page_size` default 20, maximum 100). Optional `start_date` and `end_date` (`YYYY-MM-DD`) filter on the entry's `created_at`.

#### Create Agent Knowledge

```bash
curl -X POST \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/agents/python-tutor/agent-memories/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{"content": "Our refund window is 30 days"}'
```

`content` is required and must be at least 10 characters. Entries deduplicate on a content hash: an identical entry returns `409` with `{"error": "Memory already exists"}` instead of a duplicate; a new entry returns `201` with the created object.

#### Update Agent Knowledge

```bash
curl -X PATCH \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/agents/python-tutor/agent-memories/4/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{"content": "Our refund window is 45 days"}'
```

Replaces the entry's content. If the new content collides with another of the agent's entries, returns `409`.

#### Delete Agent Knowledge

```bash
curl -X DELETE \
  "https://api.ibl.ai/api/ai-agent/orgs/acme/agents/python-tutor/agent-memories/4/" \
  -H "Authorization: Token <token>"
```

Returns `204` with no body. Everyone who chats with the agent stops receiving the entry immediately.

> **Note:** An agent-knowledge object carries `content`, `created_at`, `updated_at` and the creating user, but **no `category` and no `is_auto_generated` field** — the store is shared and manually curated, so the flag that marks an extracted memory elsewhere does not exist on it. Keep personal data out of it: anything about one individual belongs in that individual's memory, where it is scoped to its owner and deletable on request. There is no separate permission for it; the agent's own edit permission governs writes.

---

### API Endpoints Summary

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/orgs/{org}/users/{user_id}/global-memories/` | List global memories (supports filtering by session_id, content, start_date, end_date) |
| POST | `/orgs/{org}/users/{user_id}/global-memories/` | Create global memory |
| DELETE | `/orgs/{org}/users/{user_id}/global-memories/{id}/` | Delete global memory |
| GET | `/orgs/{org}/users/{user_id}/agent-memories/` | List all agent memories |
| GET | `/orgs/{org}/users/{user_id}/agents/{agent}/agent-memories/` | List agent-specific memories |
| POST | `/orgs/{org}/users/{user_id}/agents/{agent}/agent-memories/` | Create agent memory |
| PATCH | `/orgs/{org}/users/{user_id}/agents/{agent}/agent-memories/{id}/` | Update agent memory |
| DELETE | `/orgs/{org}/users/{user_id}/agents/{agent}/agent-memories/{id}/` | Delete agent memory |
| GET | `/orgs/{org}/agents/{agent}/agent-memories/` | List agent knowledge (shared memory; paged, start_date / end_date) |
| POST | `/orgs/{org}/agents/{agent}/agent-memories/` | Create agent knowledge (`409` on duplicate content) |
| PATCH | `/orgs/{org}/agents/{agent}/agent-memories/{id}/` | Update agent knowledge |
| DELETE | `/orgs/{org}/agents/{agent}/agent-memories/{id}/` | Delete agent knowledge (`204`) |
| GET | `/orgs/{org}/users/{user_id}/memsearch-settings/` | Get user settings |
| PUT | `/orgs/{org}/users/{user_id}/memsearch-settings/` | Update user settings |
| GET | `/orgs/{org}/agents/{agent}/memory-categories/` | List categories |
| POST | `/orgs/{org}/agents/{agent}/memory-categories/` | Create category |
| PATCH | `/orgs/{org}/agents/{agent}/memory-categories/{id}/` | Update category |
| DELETE | `/orgs/{org}/agents/{agent}/memory-categories/{id}/` | Deactivate category |

Note the two path shapes. Anything **about a user** is nested under `/users/{user_id}/`. Anything **belonging to the agent** — memory categories and agent knowledge — is agent-scoped under `/orgs/{org}/agents/{agent}/` and takes no `user_id`, because there is no user it is about.

---

## Technical Details

### Embedding Specifications

| Property | Value |
|----------|-------|
| Dimensions | 1536 |
| Provider | OpenAI / Azure OpenAI |
| Storage | PostgreSQL with PGVector extension |
| Search Method | Cosine Distance |

### Deduplication Thresholds

| Check | Threshold | Description |
|-------|-----------|-------------|
| Hash Match | Exact | SHA-256 content hash comparison |
| Semantic Similarity | 0.15 | Cosine distance threshold (lower = more similar) |

### Background Task Configuration

| Task | Queue | Timeout |
|------|-------|---------|
| `process_message_for_memory` | `ai_agent` | 60s soft / 90s hard |

### LLM Usage

Memory extraction uses a **cost-optimized small model** (e.g., `gpt-4o-mini`) to minimize costs while maintaining quality extraction.

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| Memories not being captured | Tenant memory not enabled | Enable in Tenant Profile → Memory tab |
| Memories not being captured | Agent memory not enabled | Enable in Agent Settings → Memory tab |
| Memories not being captured | User auto-capture disabled | User enables in their Profile settings |
| Memories not used in responses | User disabled "use memories" | User enables in their Profile settings |
| Duplicate memories appearing | Rare hash collision | Delete duplicate via API or SPA |
| Extraction taking too long | LLM provider latency | Check provider status, consider fallback |
| No categories showing | Categories not seeded | Categories auto-seed on first extraction |
