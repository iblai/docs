# Chat Metadata Pass-Through

Pass arbitrary context alongside chat messages to any AI Mentor so it can tailor responses to the user's exact situation.

---

## Overview

The chat metadata feature lets you pass arbitrary context alongside chat messages to any IBL Mentor. This context (e.g., product name, plan tier, user role) is automatically injected into the mentor's awareness, allowing it to tailor responses to the user's exact situation without the user having to explain their context manually.

**Key points:**
- The `metadata` field is a **free-form JSON object** — any key-value pairs are accepted, no enforced schema
- **Optional** — if omitted, the mentor behaves as before
- **Persistent per session** — send it once and it sticks for the entire conversation
- **Available in analytics** — stored with the conversation for reporting and data exports

---

## How It Works

```
Frontend Application
    |
    |  Sends: { prompt, metadata: { product, planTier, ... } }
    |
    v
IBL Mentor Backend
    |
    ├─ Caches metadata for the session (reused on subsequent messages)
    ├─ Persists metadata to the session record (for analytics/reporting)
    |
    v
Mentor (LLM) receives the user's message with context appended:

    "How do I set up SSO?"

    <CONTEXT METADATA> Here is additional context metadata for this conversation:
    product: Analytics
    planTier: Enterprise
    userRole: Admin
    region: EU
    </END CONTEXT METADATA>
```

The mentor sees this context and responds accordingly — e.g., answering with Enterprise-tier SSO setup steps for the Analytics product with EU data-residency considerations, rather than generic help.

---

## Sending Metadata

### WebSocket (`/ws/chat/`)

Include the `metadata` field in the JSON payload:

```javascript
ws.send(JSON.stringify({
  session_id: sessionId,
  prompt: userMessage,
  page_content: pageContent,       // optional
  metadata: {                       // optional
    product: 'Analytics',
    planTier: 'Enterprise',
    userRole: 'Admin',
    region: 'EU'
  },
  flow: { name: mentorId, tenant: tenantKey }
}));
```

### HTTP SSE (`POST /api/mentor/chat/`)

Same `metadata` field in the request body:

```http
POST /api/mentor/chat/
Content-Type: application/json

{
  "session_id": "uuid",
  "prompt": "How do I set up SSO?",
  "metadata": {
    "product": "Analytics",
    "planTier": "Enterprise",
    "userRole": "Admin",
    "region": "EU"
  },
  "flow": { "name": "mentor-slug", "tenant": "tenant-key" }
}
```

---

## Session Behavior

Metadata is **cached per session** and only needs to be sent once. Here's how it behaves across multiple messages:

| Message | `metadata` sent | What the mentor sees |
|---------|-----------------|----------------------|
| #1 | `{ product: "Analytics", planTier: "Enterprise" }` | User's prompt + Analytics / Enterprise context |
| #2 | _(not sent)_ | User's prompt + Analytics / Enterprise context (reused from #1) |
| #3 | `{ product: "Payments", planTier: "Starter" }` | User's prompt + Payments / Starter context (replaced) |
| #4 | _(not sent)_ | User's prompt + Payments / Starter context (reused from #3) |

**Rules:**
- **Send once** — metadata persists automatically for the rest of the session
- **Replace, not merge** — sending new metadata replaces the previous value entirely
- **Null or omitted = no change** — the previously cached metadata continues to be used
- **Cache TTL** — metadata is cached for 2 hours; it is also persisted to the database so it survives cache expiration

---

## Metadata Structure

The metadata field accepts **any** JSON object. There is no enforced schema — use whatever keys make sense for your application.

**Example: SaaS customer support**
```json
{
  "product": "Analytics",
  "planTier": "Enterprise",
  "userRole": "Admin",
  "region": "EU"
}
```

**Example: Corporate training platform**
```json
{
  "department": "Engineering",
  "courseId": "SEC-201",
  "courseName": "Security Fundamentals",
  "employeeLevel": "Senior"
}
```

**Example: E-commerce product advisor**
```json
{
  "category": "Laptops",
  "brand": "ThinkPad",
  "priceRange": "1000-2000",
  "customerSegment": "Business"
}
```

---

## Customizing Mentor Behavior with Metadata

The metadata is appended to the user's prompt as context. The mentor's **system prompt** determines how it uses this context. If the mentor isn't responding the way you expect based on the metadata, you can edit the system prompt to reference the metadata fields explicitly.

**Example system prompt excerpt:**
```
You are a product support assistant. When the user's context includes a
product name, tailor your answers to that specific product's features and
documentation. When planTier is provided, only suggest features available
on that plan. When region is EU, highlight GDPR and data-residency details.
```

---

## Reading Metadata from APIs

Once metadata is sent with a conversation, it's available through several APIs for analytics, reporting, and data exports.

| Endpoint | Field |
|----------|-------|
| `GET /api/ai-mentor/orgs/{org}/users/{user_id}/sessions/{session_id}/` (v1 session detail) | `client_context` in the response body |
| `GET /api/analytics/messages/details/?platform_key={platform_key}&session_id={session_id}` (v2 conversation detail) | `summary.client_context` |
| `GET /api/ai-mentor/orgs/{org}/users/{user_id}/sessions/{session_id}/tasks/{task_id}/` (chat history export) | `client_context` column in CSV |
| Analytics export (`get_chat_message_history`) | `client_context` column in DataFrame |

### Data Exports

| Export method | Where metadata appears |
|---------------|----------------------|
| CSV export (downloadable chat history) | `client_context` column |
| Chat history export task | `client_context` field per message |
| Analytics export (DataFrame) | `client_context` column |

---

## Example: Single Mentor Serving Multiple Contexts

**Scenario:** One support mentor handles questions across your entire SaaS platform, but users are on different product pages with different plans.

**Page: Analytics product — Enterprise plan**

The frontend sends:
```json
{
  "metadata": {
    "product": "Analytics",
    "planTier": "Enterprise",
    "userRole": "Admin",
    "region": "EU"
  }
}
```

**User asks:** "How do I set up SSO?"

**Mentor responds** with Enterprise-tier SSO setup steps for the Analytics product, noting EU data-residency requirements — not generic help docs.

**Page: Payments product — Starter plan**

The frontend sends:
```json
{
  "metadata": {
    "product": "Payments",
    "planTier": "Starter",
    "userRole": "Developer",
    "region": "US"
  }
}
```

**Same mentor** now responds with Starter-plan Payments integration guides, and notes that SSO requires upgrading to the Pro plan.

If the mentor's responses don't align with the expected behavior, adjust the **system prompt** to instruct the mentor on how to use the metadata fields.

---

## Architecture Reference

```
Client payload
    |
    v
BaseConsumerPayload (pydantic validation)
    |
    v
BaseLLMRunnerConsumer.process_text_data()
    |
    ├─► Redis Cache: session_{session_id}_metadata  (2-hour TTL, fast access)
    ├─► Session.metadata["client_context"]           (persistent DB storage)
    |
    v
LLMRunner.asetup_user_prompt()
    |
    ├─ 1. Add greeting instructions
    ├─ 2. Append page_content (if provided)
    └─ 3. Append metadata (formatted as key: value pairs)
    |
    v
LLM receives the composed prompt
    |
    v
Response returned to user
    |
    v
Message saved to chat history
(metadata markers remain in saved messages)
```

**Storage layers:**

| Layer | Location | Purpose |
|-------|----------|---------|
| Cache | Redis `session_{id}_metadata` | Fast access during active conversation (2h TTL) |
| Session DB | `Session.metadata["client_context"]` | Persistent storage, used by analytics APIs and exports |
| Message DB | `ChatMessageHistoryExtra.metadata` | Per-message snapshot (captures metadata at time of each message) |

**Transport:** Both WebSocket (`/ws/chat/`) and HTTP SSE (`POST /api/mentor/chat/`) share the same pipeline. The metadata field works identically on both.

---

## Notes

- The metadata structure is **generic** — not tied to any specific client's data model. Use whatever keys make sense for your application.
- Metadata is stored at the **session level**. All messages in a conversation share the same metadata unless the frontend sends an update.
- Metadata is **not stripped** from chat history (unlike `page_content`, which is stripped before saving). The `<CONTEXT METADATA>` markers remain in the stored messages.
- If the mentor's behavior doesn't reflect the metadata context as expected, update the mentor's **system prompt** to explicitly reference the metadata fields.
