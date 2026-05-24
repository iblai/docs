# Administration

<iframe width="560" height="315" src="https://www.youtube.com/embed/Y4rLO5y0mzE" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Configure MCP connectors so agents can call external MCP tools and return results inside the chat.

---

## MCP Tab Overview

- **Featured connectors:** prebuilt/default (some open-source) you can enable  
- **Custom connectors:** define your own endpoint and authentication

---

## Add a Custom Connector

1. Go to **MCP → Custom Connectors → Add**.
2. Fill the fields:
   - **Image** (optional)
   - **Name**
   - **URL** (connector/server URL)
   - **Description** (optional)
   - **Transport type**
   - **Authentication method:**
     - None
     - API Key
   - **Token type for the header** (e.g., Bearer, Basic, Token)
   - **Token value**
3. Click **Connect** to save.  
   - The connector appears in your list.
4. You can **enable/disable** any connector.  
   - Disabling prevents agents from using that MCP server in replies.

---

## Example: Connect an MCP Server with API Key

1. In the MCP provider, copy the **Access URL** and **Access Token**.
2. In **MCP → Custom Connectors**, set:
   - **Name:** e.g., Workflow MCP
   - **URL:** paste the access URL
   - **Auth:** API Key
   - **Token type:** Bearer
   - **Token value:** paste the token
3. Click **Connect**.  
   - The connector is added and can be toggled on/off.

---

## Using MCP in Chat (Demo Flow)

1. Grab a **project ID** from your MCP workspace.
2. Ask the agent, for example:
   - “List the workflows in the project `<PROJECT_ID>`.”
3. The agent lists active workflows (e.g., FAQ, webhook test, etc.).
4. Run a workflow, for example:
   - “Execute the FAQ workflow and tell me: How do I reset my password?”
5. The agent includes your question in the workflow request body and returns the answer from the MCP server.

---

## Result

Your agents can call **enabled MCP connectors** to list resources and execute workflows, returning **MCP-sourced answers** directly in the conversation.

---

# MCP Servers

Model Context Protocol (MCP) servers that expose APIs as tools for AI agents like Claude Code and Cursor.

## Available Servers

| Server | Endpoint | Description |
|--------|----------|-------------|
| Analytics | `/mcp/analytics/` | Learning analytics, agent usage, LMS metrics |
| Search | `/mcp/search/` | Course catalog and agent search (global only, personalized excluded) |
| Agent Chat | `/mcp/agent-chat/` | Chat with AI agents |
| Agent Create | `/mcp/agent-create/` | Create and manage AI agents |

## Authentication

All servers require a **Platform API Key**:

1. Log into the Admin panel
2. Navigate to Platform Settings > API Keys
3. Create and copy your API key

---

## Analytics Server

Get insights about learning activity, agent usage, and costs.

### Description (for MCP Setup)

> The iblai-analytics MCP server enables AI agents to access detailed analytics about agent-student interactions, conversation patterns, topic analysis, sentiment tracking, and cost reporting. This server provides comprehensive insights into learning activity, agent usage, user engagement metrics, LLM costs, and platform effectiveness. Key features include: learner analytics, content analytics, message and session tracking, topic analysis, financial reporting, and time-based metrics.

### Setup

```json
{
  "mcpServers": {
    "iblai-analytics": {
      "url": "https://your-instance.com/mcp/analytics/",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Api-Token YOUR_API_KEY"
      }
    }
  }
}
```

### Example Queries

Once configured, ask your AI assistant:

- "How many users are active in the acme platform?"
- "What topics are users asking about most?"
- "Show me conversation trends for the last month"
- "What are the LLM costs broken down by agent?"
- "Which users have the most chat messages?"
- "What's the average course grade?"

### Working Examples

#### Basic Health Check and User Info

```python
# Health check
ping()  # Returns: "2026-01-12T23:08:47.008677+00:00"

# Get current user
get_current_user()  # Returns: "ibl_user"

# Get user platforms
get_user_platforms()
# Returns: "User: ibl_user\nMember of platforms: ibl, main, ...\nAdmin of platforms: ibl, main, ..."

# Count users in platform
count_users_in_platform(platform="ibl")  # Returns: 8
```

#### Learner Analytics

```python
# Get learner analytics for a specific user
get_learner_analytics(
    username="ibl_user",
    platform_key="ibl"
)
# Returns: User summary with time spent, enrollments, credentials, etc.

# Get learner list for a platform
get_learner_list(
    platform_key="ibl",
    limit=5
)
# Returns: Paginated list of learners with metrics

# Get detailed learner information
get_learner_details(
    username="ibl_user",
    platform_key="ibl"
)
# Returns: Consolidated learner analytics across catalog, agent, and credential data
```

#### Content Analytics

```python
# Get content analytics (requires platform_key)
get_content_analytics(
    metric="courses",
    platform_key="ibl"
)
# Returns: Course analytics with enrollments, time spent, completion rates, etc.

# Get detailed content information
get_content_details(
    content_id="course-v1:Academy+S-IB-50+v1",
    metric="course",
    platform_key="ibl"
)
# Returns: Detailed course information with learner breakdowns
```

**Important Notes:**
- Most analytics endpoints **require** `platform_key` parameter
- `get_web_analytics_time_spent_per_user` requires admin user privileges
- Use `metric` parameter to specify content type: `"course"`, `"courses"`, `"program"`, `"programs"`, `"pathway"`, `"pathways"`, `"skill"`, or `"skills"`

---

## Search Server

> **⚠️ Temporarily Disabled**: The Search server is currently disabled while we clean up naming conventions. It will be re-enabled in a future update.

Search for courses and agents in the catalog.

### Description (for MCP Setup)

> The iblai-search MCP server enables AI agents to search for courses, programs, pathways, and agents in the learning catalog. This server provides global search capabilities for discovering educational content and AI agents. Key features include: catalog search across courses, programs, pathways and skills; agent search to find AI tutors by topic or expertise; and course recommendations based on user preferences.

### Setup

```json
{
  "mcpServers": {
    "iblai-search": {
      "url": "https://your-instance.com/mcp/search/",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Api-Token YOUR_API_KEY"
      }
    }
  }
}
```

### Example Queries

- "Find courses about machine learning"
- "Search for agents that teach Python"
- "What courses would you recommend for a beginner?"
- "Show me the most popular courses in the catalog"

### Working Examples

#### Health Check

```python
# Health check
ping()  # Returns: "2026-01-12T23:14:26.318962+00:00"
```

#### Catalog Search

```python
# Search catalog for courses, programs, pathways, skills
get_catalog_search(
    query="test",
    limit=5
)
# Returns: Search results with courses, programs, pathways
# Includes facets for filtering (topics, subjects, tags, etc.)
```

#### Agent Search

```python
# Search for agents (requires platform_key for authenticated requests)
get_mentor_search(
    query="test",
    limit=5,
    platform_key="ibl"
)
# Returns: List of agents matching the query with metadata
```

**Important Notes:**
- `get_mentor_search` **requires** `platform_key` parameter for authenticated requests
- Use `platform_key` or `tenant` parameter (both serve the same purpose)
- Search results include pagination information

#### Recommendations

```python
# Get course recommendations
get_recommendations(
    platform_key="ibl",
    limit=3
)
# Returns: Personalized course recommendations based on RAG search
# Can specify recommendation_type: "agents", "courses", "programs", "resources", "pathways"
```

---

## Agent Create Server

Create and manage AI agents, including CRUD operations, forking, and configuration.

### Description (for MCP Setup)

> The iblai-agent-create MCP server enables AI agents to create, configure, and manage AI agents. This server provides full agent lifecycle management including creation from templates, settings configuration, and document training. Key features include: create agents from templates; configure LLM settings (provider, model, temperature); manage display settings and feature flags; train agents with documents, URLs, or text content; and update agent prompts and permissions.

### Setup

```json
{
  "mcpServers": {
    "iblai-agent-create": {
      "url": "https://your-instance.com/mcp/agent-create/",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Api-Token YOUR_API_KEY"
      }
    }
  }
}
```

### Example Queries

- "Create a new agent for data science"
- "List all agents in the platform"
- "Update agent settings for agent ID 123"
- "Fork agent ID 456 with new name"
- "Show me available agent templates"
- "What categories are available for agents?"

### Working Examples

#### Health Check

```python
# Health check
ping()  # Returns: "2026-01-12T22:57:37.066570+00:00"
```

#### Get Agent Settings

```python
# Retrieve agent settings
get_mentor_settings(
    org="ibl",
    user_id="ibl_user",
    agent="ad2be335-5afa-4a9e-9298-8273b3d94e10"
)
# Returns: Complete agent configuration including:
# - Display settings (theme, colors, images)
# - LLM configuration (provider, model, temperature)
# - Feature flags (image generation, web browsing, code interpreter)
# - Prompts (system, proactive, study mode)
# - Visibility and permissions
```

#### Create Agent from Template

```python
# Create a new agent from a template
post_mentor_with_settings(
    org="ibl",
    user_id="ibl_user",
    template_name="ai-agent",
    new_mentor_name="test-agent-123"
)
# Returns: Created agent object with unique_id, settings, and configuration
# Optional parameters: display_name, description, system_prompt, llm_provider, etc.
```

#### Update Agent Settings

```python
# Update existing agent settings
put_mentor_settings(
    org="ibl",
    user_id="ibl_user",
    agent="ad2be335-5afa-4a9e-9298-8273b3d94e10",
    mentor_description="Updated description for testing"
)
# Returns: Updated agent settings object
# Can update: mentor_name, display_name, system_prompt, llm_provider,
# enable_image_generation, enable_web_browsing, categories, types, subjects, etc.
```

#### Train Documents

```python
# Train a document directly (for smaller documents)
post_retriever_train(
    org="ibl",
    user_id="ibl_user",
    pathway="ad2be335-5afa-4a9e-9298-8273b3d94e10",  # Use mentor_id as pathway
    url="https://example.com/test-document"
)
# Returns: {"detail": "Document trained successfully"}

# Train a document through worker process (for larger documents)
post_train_document(
    org="ibl",
    user_id="ibl_user",
    type="url",
    pathway="ad2be335-5afa-4a9e-9298-8273b3d94e10",  # Use mentor_id as pathway
    url="https://example.com/test"
)
# Returns: Task confirmation or error message
```

**Important Notes:**
- For `post_train_document` and `post_retriever_train`, use the **mentor_id** as the `pathway` parameter
- The `pathway` parameter must be a valid agent unique ID (UUID format)
- `post_train_document` supports multiple types: `"file"`, `"url"`, `"text"` (for Wikipedia), etc.
- If URL is not accessible, you'll get: `"We couldn't reach the website. It may be offline or blocking access."`

---

## Agent Chat Server

Have conversations with AI agents. Requires specifying which agent to use.

### Description (for MCP Setup)

> The iblai-agent-chat MCP server enables AI agents to have conversations with configured AI agents. This server acts as a bridge to interact with specialized agents that have been trained on specific topics or documents. The agent is specified via the X-Agent-Unique-Id header, and responses are based on the agent's system prompt, LLM configuration, and trained knowledge base.

### Setup

```json
{
  "mcpServers": {
    "iblai-agent": {
      "url": "https://your-instance.com/mcp/agent-chat/",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Api-Token YOUR_API_KEY",
        "X-Agent-Unique-Id": "YOUR_MENTOR_ID"
      }
    }
  }
}
```

### Example Queries

- "Explain how photosynthesis works"
- "Help me understand quadratic equations"
- "What are the key events of World War II?"
- "Can you review this code and suggest improvements?"

### Working Examples

#### Chat with Agent

```python
# Get a response from an agent
get_mentor_response(
    prompt="hello"
)
# Returns: "Hello! How can I assist you today?"

# More complex query
get_mentor_response(
    prompt="Can you explain machine learning in simple terms?"
)
# Returns: Detailed explanation from the agent based on their configuration
```

**Important Notes:**
- The agent is specified via the `X-Agent-Unique-Id` header in the MCP server configuration
- The agent's response depends on their system prompt, LLM configuration, and any trained documents
- Simple prompts like "hello" work well for testing connectivity

---

## Full Configuration

To use all available servers together:

```json
{
  "mcpServers": {
    "iblai-analytics": {
      "url": "https://your-instance.com/mcp/analytics/",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Api-Token YOUR_API_KEY"
      }
    },
    "iblai-agent": {
      "url": "https://your-instance.com/mcp/agent-chat/",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Api-Token YOUR_API_KEY",
        "X-Agent-Unique-Id": "YOUR_MENTOR_ID"
      }
    },
    "iblai-agent-create": {
      "url": "https://your-instance.com/mcp/agent-create/",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Api-Token YOUR_API_KEY"
      }
    },
    "iblai-search": {
      "url": "https://your-instance.com/mcp/search/",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Api-Token YOUR_API_KEY"
      }
    }
  }
}
```

> **Note**: The `iblai-search` server excludes personalized search tools (personalized agent search and personalized catalog search).

Save this to:
- **Claude Code**: `~/.claude/claude_desktop_config.json` or `.mcp.json` in your project
- **Cursor**: Add to Settings > Features > MCP Servers

---

## Tool Generation and Endpoints

This section documents the endpoints used by each MCP server and how tool names are generated.

### Tool Name Generation Format

Tool names follow the pattern: **`{HTTP_METHOD}_{usable_name}`**

Where:
- **`{HTTP_METHOD}`**: The HTTP method in lowercase (e.g., `get`, `post`, `put`, `patch`, `delete`)
- **`{usable_name}`**: Derived from the URL pattern name with transformations:
  - Hyphens converted to underscores
  - Redundant prefixes removed (`apis_`, `ai_analytics_`, `ai_mentor_`, `v2_`, `trigram_`)
  - Section prefixes removed (for analytics: `overview_`, `audience_`, etc.)
  - Common words abbreviated (e.g., `completion` → `compl`, `enrollment` → `enroll`)

**Example**: URL pattern name `ai_mentor_orgs_users_mentor_with_settings` → Tool name `post_mentor_with_settings`

---

### iblai-agent (Agent Chat Server)

**Custom Tool** (not auto-generated from endpoints):

| Tool Name | Endpoint | Method | Description |
|-----------|----------|--------|-------------|
| `get_mentor_response` | N/A (custom implementation) | N/A | Get a response from an agent. Requires `X-Agent-Unique-Id` header. |

**Note**: This is a custom tool that directly calls the LLM service, not generated from an API endpoint.

---

### iblai-agent-create (Agent Create Server)

**Endpoints** (auto-generated from API patterns):

| Tool Name | Endpoint | Method | URL Pattern Name |
|-----------|----------|--------|------------------|
| `post_mentor_with_settings` | `/api/ai-agent/orgs/{org}/users/{user_id}/agent-with-settings/` | POST | `agent-with-settings` |
| `get_mentor_settings` | `/api/ai-agent/orgs/{org}/users/{user_id}/agents/{agent}/settings/` | GET | `agent-settings` |
| `put_mentor_settings` | `/api/ai-agent/orgs/{org}/users/{user_id}/agents/{agent}/settings/` | PUT | `agent-settings` |
| `post_train_document` | `/api/ai-index/orgs/{org}/users/{user_id}/documents/train/` | POST | `ai_index_orgs_users_documents_train` |
| `post_retriever_train` | `/api/ai-index/orgs/{org}/users/{user_id}/documents/train/retriever/` | POST | `ai_index_orgs_users_documents_train_retriever` |
| `ping` | N/A (custom tool) | N/A | Health check tool |

**Tool Name Generation Examples**:
- `agent-with-settings` → `post_mentor_with_settings` (hyphen → underscore, method prefix added)
- `agent-settings` → `get_mentor_settings` / `put_mentor_settings` (different methods create different tools)
- `ai_index_orgs_users_documents_train` → `post_train_document` (prefixes removed, simplified)

---

### iblai-search (Search Server)

**Endpoints** (auto-generated from search patterns, excluding personalized):

| Tool Name | Endpoint | Method | Description |
|-----------|----------|--------|-------------|
| `get_mentor_search` | `/api/ai-search/agents/` | GET | Global agent search (personalized excluded) |
| `get_catalog_search` | `/api/search/catalog/` | GET | Global catalog search (personalized excluded) |
| `get_recommendations` | `/api/ai-search/recommendations/` | GET | Course recommendations |
| `ping` | N/A (custom tool) | N/A | Health check tool |

**Excluded Tools** (personalized endpoints):
- `get_personalized_mentors` - Personalized agent recommendations
- `get_personalized_catalog_search` - Personalized catalog search

**Tool Name Generation**:
- URL pattern names are cleaned by removing redundant prefixes
- Only GET methods are included (`only_get=True` in registration)
- Personalized endpoints are explicitly filtered out

**Custom Tools** (not auto-generated):
- `ping_search`: Health check

---

### iblai-analytics (Analytics Server)

**Endpoints** (auto-generated from `/api/analytics/` namespace):

All endpoints under `/api/analytics/` are automatically included. Examples:

| Tool Name | Endpoint | Method | Description |
|-----------|----------|--------|-------------|
| `get_content_analytics` | `/api/analytics/content/` | GET | Get content analytics |
| `get_content_details` | `/api/analytics/content/details/{content_id}/` | GET | Get detailed content analytics |
| `get_learner_analytics` | `/api/analytics/learners/` | GET | Get learner analytics |
| `get_learner_list` | `/api/analytics/learners/list/` | GET | Get list of learners |
| `get_learner_details` | `/api/analytics/learner/details/` | GET | Get detailed learner analytics |
| `get_web_analytics_time_spent_per_user` | `/api/analytics/time-spent/user/` | GET | Get time spent per user |
| `get_financial` | `/api/analytics/financial/` | GET | Get financial analytics |
| `get_financial_details` | `/api/analytics/financial/details/` | GET | Get detailed financial analytics |
| `get_financial_invoice` | `/api/analytics/financial/invoice/` | GET | Get invoice data |
| `get_messages` | `/api/analytics/messages/` | GET | Get message analytics |
| `get_sessions` | `/api/analytics/sessions/` | GET | Get session analytics |
| `get_ratings` | `/api/analytics/ratings/` | GET | Get rating analytics |
| `get_topics` | `/api/analytics/topics/` | GET | Get topic analytics |
| `get_time` | `/api/analytics/time/` | GET | Get time-based analytics |
| `ping` | N/A (custom tool) | N/A | Health check tool |
| `count_users_in_platform` | N/A (custom tool) | N/A | Count active users in platform |
| `get_current_user` | N/A (custom tool) | N/A | Get current authenticated user |
| `get_user_platforms` | N/A (custom tool) | N/A | Get user's platform memberships |

**Tool Name Generation**:
- URL pattern names are cleaned by removing redundant prefixes and applying abbreviations
- Only GET methods are included (`only_get=True` in registration)
- All endpoints starting with `/api/analytics/` are automatically included

**Custom Tools** (not auto-generated):
- `ping`: Health check
- `count_users_in_platform`: Direct database query
- `get_current_user`: Authentication helper
- `get_user_platforms`: Authentication helper

---

## Troubleshooting

### Authentication Issues

**"Unauthorized" errors**: Check your API key is correct and hasn't expired.

**"Could not authenticate agent"**: Verify the `X-Agent-Unique-Id` header is set correctly.

**Connection issues**: Ensure the instance URL is correct and accessible.

### Common Parameter Errors

**"platform_key is required"** (Analytics & Search):
- Most analytics endpoints require the `platform_key` parameter
- Example: `get_content_analytics(metric="courses", platform_key="ibl")`
- The `platform_key` should match your organization/platform identifier

**"Invalid parameters"** (Search):
- `get_mentor_search` requires `platform_key` for authenticated requests
- Use either `platform_key` or `tenant` parameter (both serve the same purpose)

**"Document pathway is not a valid agent unique id"** (Agent Create):
- When using `post_train_document` or `post_retriever_train`, the `pathway` parameter must be a valid agent UUID
- Use the agent's unique ID (not the agent name or slug)
- Example: `pathway="ad2be335-5afa-4a9e-9298-8273b3d94e10"`

**"We couldn't reach the website"** (Agent Create):
- When training documents from URLs, ensure the URL is publicly accessible
- The URL may be blocked, require authentication, or be offline
- Verify the URL works in a browser before using it

### Permission Errors

**"Requires admin user"** (Analytics):
- Some endpoints like `get_web_analytics_time_spent_per_user` require admin privileges
- Ensure your API key has admin permissions for the platform

### Best Practices

1. **Always include `platform_key`** for analytics and search operations
2. **Use agent UUIDs** (not names) for pathway parameters in document training
3. **Test with simple prompts first** (e.g., "hello") before complex queries
4. **Check URL accessibility** before training documents from external sources
5. **Use health check tools** (`ping`) to verify server connectivity
