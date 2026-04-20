# Memory System

The Memory System enables AI mentors to remember information about users across conversations. It stores user preferences, learning progress, knowledge gaps, and personal context, making interactions more personalized and contextual.

---

## Overview

The memory system supports two scopes of memory:

| Memory Type | Scope | Description |
|-------------|-------|-------------|
| **Global User Memories** | All mentors | Facts about the user that apply everywhere (name, profession, preferences) |
| **Mentor-Specific Memories** | Single mentor | Context specific to interactions with a particular mentor |

**Default Memory Categories (Mentor-Specific):**

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

1. **Extraction path (write):** A Chat Session feeds into a background Celery Task containing the Extraction Service, which processes conversations and writes to the Memory Store (PGVector) — storing global memories, mentor memories, and their embeddings.
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
│ - Mentor enabled?│     └─────────────────┘
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
Mentor-specific:
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
│ Check mentor     │────▶│ NO INJECTION    │
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
│ - Top 5 mentor   │
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
## User Information
- The user is a software engineer with 5 years of experience
- The user prefers visual explanations with diagrams

## Relevant Context from Previous Conversations
- [Knowledge Gaps] The user struggled with understanding recursion
- [Learning Goals] The user wants to master system design patterns
- [Preferences] The user prefers Python code examples over pseudocode
```

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
│  │  Mentor Settings → Memory Tab                        │   │
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

> **Note:** This is the master switch. Memory features will not work for any mentor or user until this is enabled.

---

### Enabling Memory for a Mentor

Each mentor can have memory features enabled or disabled individually:

1. Navigate to **Mentor Settings** for the specific mentor
2. Select the **Memory** tab
3. Toggle memory to **enabled**

Once enabled, the mentor will:
- Automatically extract memories from conversations
- Use stored memories to personalize responses

**Managing Mentor Memory Categories:**

From the Mentor Settings → Memory tab, administrators can:
- View default memory categories
- Create custom categories
- Edit category names and descriptions
- Deactivate categories (memories in that category will no longer be extracted)

**Viewing User Memories for a Mentor:**

The Mentor Settings → Memory tab also displays all memories stored for users interacting with that mentor, organized by category.

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

> **Note:** Users can disable memory features entirely for privacy, even if the tenant and mentor have memory enabled.

---

### Managing Global User Memories

Global memories are facts about the user that apply across all mentors (e.g., "The user is a software engineer").

**Location:** User Profile page → Global Memories section

**User Actions:**

| Action | Description |
|--------|-------------|
| **View memories** | See all automatically captured and manually added global memories |
| **Add memory** | Manually add a new global memory |
| **Delete memory** | Remove a memory that is no longer relevant or accurate |

---

### Managing Mentor-Specific Memories

Mentor memories are specific to a user's interactions with a particular mentor.

**Location:** Mentor Settings page → Memory tab → User Memories section

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

Base URL: `/api/ai-mentor/`

### Global Memories API

**Endpoints:** `/orgs/{org}/users/{user_id}/global-memories/`

#### List Global Memories

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/global-memories/" \
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
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/global-memories/?session_id=abc-123-def" \
  -H "Authorization: Token <token>"
```

**Search by content:**
```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/global-memories/?content=software%20engineer" \
  -H "Authorization: Token <token>"
```

**Filter by date range:**
```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/global-memories/?start_date=2024-01-01&end_date=2024-01-31" \
  -H "Authorization: Token <token>"
```

**Combined filters:**
```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/global-memories/?content=python&start_date=2024-01-01" \
  -H "Authorization: Token <token>"
```

#### Create Global Memory

```bash
curl -X POST \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/global-memories/" \
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
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/global-memories/3/" \
  -H "Authorization: Token <token>"
```

**Response:** `204 No Content`

---

### Mentor Memories API

**Endpoints:** `/orgs/{org}/users/{user_id}/mentors/{mentor_id}/mentor-memories/`

#### List All Mentor Memories (All Mentors)

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/mentor-memories/" \
  -H "Authorization: Token <token>"
```

**Response:**
```json
{
  "count": 3,
  "results": [
    {
      "id": 1,
      "mentor": "python-tutor",
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
      "mentor": "python-tutor",
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

#### List Memories for Specific Mentor

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/mentors/python-tutor/mentor-memories/" \
  -H "Authorization: Token <token>"
```

#### Filter by Category

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/mentors/python-tutor/mentor-memories/?category=knowledge_gaps" \
  -H "Authorization: Token <token>"
```

#### Create Mentor Memory

```bash
curl -X POST \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/mentors/python-tutor/mentor-memories/" \
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
  "mentor": "python-tutor",
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

#### Update Mentor Memory

```bash
curl -X PATCH \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/mentors/python-tutor/mentor-memories/4/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "The user completed Python basics and intermediate courses"
  }'
```

#### Delete Mentor Memory

```bash
curl -X DELETE \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/mentors/python-tutor/mentor-memories/4/" \
  -H "Authorization: Token <token>"
```

**Response:** `204 No Content`

---

### User Memory Settings API

**Endpoints:** `/orgs/{org}/users/{user_id}/memsearch-settings/`

#### Get User Settings

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/memsearch-settings/" \
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
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/users/user123/memsearch-settings/" \
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

**Endpoints:** `/orgs/{org}/mentors/{mentor_id}/memory-categories/`

#### List Memory Categories

```bash
curl -X GET \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/mentors/python-tutor/memory-categories/" \
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
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/mentors/python-tutor/memory-categories/" \
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
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/mentors/python-tutor/memory-categories/6/" \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Project ideas and coding challenges the user wants to try"
  }'
```

#### Deactivate Category

```bash
curl -X DELETE \
  "https://api.ibl.ai/api/ai-mentor/orgs/acme/mentors/python-tutor/memory-categories/6/" \
  -H "Authorization: Token <token>"
```

> **Note:** Deleting a category deactivates it (`is_active: false`). Existing memories in that category are preserved but no new memories will be extracted for it.

---

### API Endpoints Summary

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/orgs/{org}/users/{user_id}/global-memories/` | List global memories (supports filtering by session_id, content, start_date, end_date) |
| POST | `/orgs/{org}/users/{user_id}/global-memories/` | Create global memory |
| DELETE | `/orgs/{org}/users/{user_id}/global-memories/{id}/` | Delete global memory |
| GET | `/orgs/{org}/users/{user_id}/mentor-memories/` | List all mentor memories |
| GET | `/orgs/{org}/users/{user_id}/mentors/{mentor}/mentor-memories/` | List mentor-specific memories |
| POST | `/orgs/{org}/users/{user_id}/mentors/{mentor}/mentor-memories/` | Create mentor memory |
| PATCH | `/orgs/{org}/users/{user_id}/mentors/{mentor}/mentor-memories/{id}/` | Update mentor memory |
| DELETE | `/orgs/{org}/users/{user_id}/mentors/{mentor}/mentor-memories/{id}/` | Delete mentor memory |
| GET | `/orgs/{org}/users/{user_id}/memsearch-settings/` | Get user settings |
| PUT | `/orgs/{org}/users/{user_id}/memsearch-settings/` | Update user settings |
| GET | `/orgs/{org}/mentors/{mentor}/memory-categories/` | List categories |
| POST | `/orgs/{org}/mentors/{mentor}/memory-categories/` | Create category |
| PATCH | `/orgs/{org}/mentors/{mentor}/memory-categories/{id}/` | Update category |
| DELETE | `/orgs/{org}/mentors/{mentor}/memory-categories/{id}/` | Deactivate category |

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
| Memories not being captured | Mentor memory not enabled | Enable in Mentor Settings → Memory tab |
| Memories not being captured | User auto-capture disabled | User enables in their Profile settings |
| Memories not used in responses | User disabled "use memories" | User enables in their Profile settings |
| Duplicate memories appearing | Rare hash collision | Delete duplicate via API or SPA |
| Extraction taking too long | LLM provider latency | Check provider status, consider fallback |
| No categories showing | Categories not seeded | Categories auto-seed on first extraction |
