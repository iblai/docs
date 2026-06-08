# IBL CRM — Developer Documentation

> **Base URL:** `/api/crm/`
> **Authentication:** `Authorization: Token YOUR_ACCESS_TOKEN`
> **API Version:** v1

---

## 1. Introduction

The IBL CRM is a Platform-scoped surface for managing people, organizations, sales pipelines, deals, and follow-up activities alongside the rest of your IBL platform. Every record — every person, organization, pipeline, deal, activity, and tag — belongs to a single Platform and is isolated from other Platforms by the access token used to call the API. The CRM is not a standalone system: it shares identity with the rest of your platform, so when a person you previously captured as a lead eventually signs up with a matching email address, their CRM record is automatically linked to their new Platform user account.

### Key Capabilities

- People and organizations with lifecycle stage tracking (`lead`, `qualified`, `opportunity`, `customer`, `churned`)
- Multi-pipeline deal flow with named stages, per-stage win probability, and explicit won/lost terminal states
- Activity log for calls, meetings, notes, and tasks with scheduling, reminders, and mark-done semantics
- Cross-entity tagging that spans people, organizations, and deals with server-validated hex colors
- Automatic person-to-user linking when a Platform signup matches an existing CRM person by email
- Notifications on key CRM events with configurable per-Platform recipient lists

### Who Should Read This

This documentation is written for Platform administrators and developers building integrations against `/api/crm/*` — sales dashboards, lead-capture forms, kanban boards, activity timelines, and any other surface that reads from or writes to the IBL CRM.

### How the CRM Fits Your Platform

```mermaid
flowchart LR
    App([Your Application]) --> API[CRM API<br/>/api/crm/]
    API --> PO[Persons<br/>Organizations]
    API --> DP[Deals<br/>Pipelines]
    API --> AT[Activities<br/>Tags]
    API --> N[Notifications]
    Users([Platform Users]) -.->|email match| PO
```

### Table of Contents

1. [Introduction](#1-introduction)
2. [System Overview](#2-system-overview)
3. [Authentication](#3-authentication)
4. [Quickstart: Capture a Lead, Open a Deal, Close It](#4-quickstart-capture-a-lead-open-a-deal-close-it)
5. [Core Concepts](#5-core-concepts)
6. [Resource Reference](#6-resource-reference)
7. [API Reference](#7-api-reference)
   - [7.1 Persons](#71-persons)
   - [7.2 Organizations](#72-organizations)
   - [7.3 Pipelines & Stages](#73-pipelines--stages)
   - [7.4 Lead Sources](#74-lead-sources)
   - [7.5 Deals](#75-deals)
   - [7.6 Activities](#76-activities)
   - [7.7 Tags](#77-tags)
8. [Person Onboarding (Link, Invite, Merge)](#8-person-onboarding-link-invite-merge)
9. [Deal Lifecycle](#9-deal-lifecycle)
10. [Activity Timeline & Auto-Records](#10-activity-timeline--auto-records)
11. [Tagging](#11-tagging)
12. [Notifications](#12-notifications)
13. [Filtering & Pagination](#13-filtering--pagination)
14. [Roles & Permissions](#14-roles--permissions)
15. [Error Reference](#15-error-reference)
16. [Best Practices](#16-best-practices)

---
## 2. System Overview

The CRM exposes two interaction surfaces. The **write-side** accepts `POST`, `PATCH`, and `DELETE` calls to create people, organizations, deals, activities, and tags, and to drive deals through pipeline stages. The **read-side** accepts `GET` calls with rich filtering — by lifecycle stage, owner, pipeline, stage, tags, scheduling windows, metadata keys, and free-form date ranges. Both surfaces share one rule: every request is scoped to the Platform behind the token. You cannot read across Platforms, you cannot write across Platforms, and a record that lives on another Platform returns `404` rather than `403` so existence never leaks.

Inside that scope, the API is request/response. Each call validates, authorizes, persists, and returns the resulting object. Some calls trigger additional work — an audit row, a notification — but that work is wired so it never blocks the response and never fires unless the underlying write actually committed.

### Side Effects

Three classes of write trigger automatic follow-up work:

**Creating a person** fires a `CRM_PERSON_CREATED` notification. This covers people created through the API, the admin, and bulk import — every code path that produces a new person row ends with the same notification.

**Moving a deal between stages** does two things. First, it writes an audit `Activity` row capturing the transition (type `note`, marked done, titled `"Stage changed"`, with the from-stage and to-stage display names in the comment). Second, it fires a `CRM_DEAL_STAGE_CHANGED` notification. This applies to all three transition endpoints — `move-stage`, `won`, and `lost`. Transitions that resolve to the deal's current stage are suppressed: no write, no audit row, no notification.

**Linking a person to a Platform user** fires a `CRM_PERSON_LINKED_TO_USER` notification. The link can happen explicitly through the `link-user` endpoint, or implicitly when someone whose email matches an existing person record signs up for a Platform account.

Notifications are dispatched asynchronously **after the writing transaction commits**. A write that rolls back — because validation failed, a database constraint tripped, or the request errored — produces no notification. By the time a recipient sees the email or push entry, the underlying record is durably on disk.

### Request to Side-Effect Flow

```mermaid
flowchart TD
    R[REST request] --> V[Validation and RBAC]
    V --> W[Write to data store]
    W --> S{Triggers?}
    S -->|Person created| N1[Notification dispatched]
    S -->|Deal moved stage| A[Audit Activity written]
    A --> N2[Notification dispatched]
    S -->|Person linked to user| N3[Notification dispatched]
    S -->|None| END[Response returned]
    N1 --> END
    N2 --> END
    N3 --> END
```

### Deal Status State Machine

Every deal carries a `status` of `open`, `won`, or `lost`. New deals start `open`. Calls to `move-stage` keep the deal `open` as long as the destination stage is non-terminal. Stages flagged `is_won` close the deal as `won` and stamp `closed_at`; stages flagged `is_lost` do the same and close it as `lost`. The `won` and `lost` action endpoints are shortcuts that resolve to the first terminal stage of the matching kind. A closed deal is not frozen — moving it back to a non-terminal stage reopens it, clears `closed_at`, and the status returns to `open`. Status and `closed_at` are server-managed; `PATCH` attempts on those fields are rejected with `400`.

```mermaid
stateDiagram-v2
    [*] --> open: Deal created
    open --> open: move-stage to non-terminal stage
    open --> won: move-stage to is_won stage / won action
    open --> lost: move-stage to is_lost stage / lost action
    won --> open: move-stage back to non-terminal stage
    lost --> open: move-stage back to non-terminal stage
    won --> [*]
    lost --> [*]
```

### What You Will NOT See

The CRM does not expose queues, workers, schedulers, or any background-service control plane. There is no endpoint to enqueue a job, inspect a task, or replay a failed delivery. The data store is hidden behind the API — there are no database connection strings, raw SQL hooks, or schema introspection endpoints in the public surface. The only things you interact with are REST calls and the notifications that arrive in recipients' inboxes and push channels. Everything else is an implementation detail and is free to change without notice.

---
## 3. Authentication

Every CRM endpoint requires a Platform-scoped access token supplied in the `Authorization` header:

```
Authorization: Token YOUR_ACCESS_TOKEN
```

Tokens are obtained from the standard Platform authentication endpoint and are bound to a single Platform at the moment of issue. That binding is the source of truth for which CRM data the caller can see — there is no `?platform_key=` query parameter on consumer endpoints, and supplying one will not change the Platform a request resolves to.

### Example Request

```bash
curl -X GET https://platform.iblai.app/api/crm/persons/ \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Accept: application/json"
```

The same header form applies to every method (`GET`, `POST`, `PATCH`, `DELETE`) and every CRM resource. No additional auth parameters are required.

### Platform Scope

Tokens bind to exactly one Platform. Every CRM record — persons, organizations, deals, activities, tags, pipelines, stages, lead sources — is scoped to a Platform, and the API filters reads and writes to the Platform attached to your token.

If you request a resource that belongs to a different Platform, the API returns `404 Not Found`, never `403 Forbidden`. This is intentional: returning `403` would leak the existence of cross-Platform records. As far as your token is concerned, records on other Platforms simply do not exist.

The same rule applies on writes. Attempting to `POST` or `PATCH` a record onto a Platform that is not yours will fail as if the parent object were missing.

### Permission Levels

CRM access is governed by four roles. The table below is the high-level summary — the full action-by-action matrix lives in Section 14 (RBAC Reference).

| Action | Required Role |
|---|---|
| Read everything | CRM Viewer |
| Day-to-day work on people, organizations, deals, activities, tags | CRM User |
| Pipeline / stage / lead-source administration and invitations | CRM Manager |
| Read people and send invitations only | CRM Inviter |

Roles are seeded automatically on each Platform the first time the CRM is provisioned, so there is nothing to install or configure before assigning them. Assign roles to users through the standard role-management surface on the Platform — the CRM does not expose its own role-assignment endpoints.

A single user may hold more than one role; effective permissions are the union of all roles held.

### Failure Modes

| Status | Meaning |
|---|---|
| `401 Unauthorized` | The `Authorization` header is missing, malformed, or carries an expired/revoked token. |
| `403 Forbidden` | The token is valid but the caller's roles do not grant the required CRM permission for this action. Also returned when a service-account API key is presented for a Platform other than the one the key is scoped to. |
| `404 Not Found` | The requested resource does not exist on your Platform. This is also the response for resources that exist on a different Platform — existence is never leaked. |

A `403` always means "you are authenticated but not allowed"; a `404` always means "as far as your token is concerned, this record is not here." Treat the two as distinct in client code: retrying a `403` will not help, but a `404` on a record you just created usually means you are pointing at the wrong Platform.

### Service Accounts

Service-account tokens (Platform API keys) work against the same CRM endpoints as user tokens, with the same header form:

```
Authorization: Token YOUR_PLATFORM_API_KEY
```

A Platform API key is scoped to exactly one Platform at the time it is issued. Any request that resolves to a different Platform — for example, attempting to operate on records owned by another Platform — returns `403 Forbidden`. Use service-account keys for server-to-server integrations, ETL jobs, and back-office automation; use user tokens for anything driven by an end user in a browser session.

Service accounts are subject to the same RBAC matrix as users. The role attached to the service account determines which CRM verbs it can invoke.

### Security

- Store tokens server-side. Do not embed them in browser bundles, mobile binaries, or any client the end user controls.
- Never commit tokens to source control. Use environment variables, a secrets manager, or your platform's standard secret-injection mechanism.
- Treat a token like a password: rotate on suspected compromise, scope to the minimum role required, and prefer short-lived user tokens over long-lived ones where the workflow allows it.
- For service-account keys, restrict outbound network egress so the key can only be used from your infrastructure, and audit usage through the Platform's standard access logs.

A leaked token grants the holder every CRM permission the underlying account holds, on every record in that Platform. There is no per-record sharing model that limits the blast radius.

---
## 4. Quickstart: Capture a Lead, Open a Deal, Close It

End-to-end happy path in five steps. By the end you will have created a person, looked up your default pipeline, opened a deal in the `new` stage, advanced it through the pipeline, and closed it as won — exercising the full state machine the rest of this guide builds on.

### Prerequisites

Before you start, make sure you have:

- **An API token** issued for your Platform. All requests in this section use the `Authorization: Token <your-token>` header (see Section 3).
- **A seeded Platform.** On Platform creation, the CRM auto-seeds:
  - **One default Pipeline** with `code="default"` (`is_default=true`, `rotten_days=30`).
  - **Six stages** on that Pipeline, referenced by stable `code`:
    | `code`        | `name`        | `probability` | `is_won` | `is_lost` |
    |---------------|---------------|---------------|----------|-----------|
    | `new`         | New           | 10            | false    | false     |
    | `qualified`   | Qualified     | 25            | false    | false     |
    | `proposal`    | Proposal      | 50            | false    | false     |
    | `negotiation` | Negotiation   | 75            | false    | false     |
    | `won`         | Won           | 100           | true     | false     |
    | `lost`        | Lost          | 0             | false    | true      |
  - **Four lead sources**: `web`, `referral`, `cold_call`, `advertisement`.

You do not need to create any of these yourself — they are present on every Platform. You can edit, rename, or add to them later (see Section 9).

---

### Step 1: Create a person

A **Person** is the human record at the center of every deal. The only required field is `name`. `primary_email` is optional but recommended — it powers the automatic link to a Platform user when emails match. `organization` is optional. We will leave `lifecycle_stage` at its default of `lead`.

```bash
curl -X POST https://platform.iblai.app/api/crm/persons/ \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Chen",
    "primary_email": "alice.chen@acme.example",
    "job_title": "VP Engineering",
    "lifecycle_stage": "lead"
  }'
```

**Response — `201 Created`:**

```json
{
  "id": "8f2a1c4d-9b3e-4a7f-9c2d-1e5b6a7f8c9d",
  "platform": 1,
  "name": "Alice Chen",
  "primary_email": "alice.chen@acme.example",
  "emails": [],
  "contact_numbers": [],
  "job_title": "VP Engineering",
  "organization": null,
  "owner": null,
  "platform_user": null,
  "lifecycle_stage": "lead",
  "unique_id": "",
  "active": true,
  "tags": [],
  "metadata": {},
  "created_at": "2026-06-04T14:22:11.482913Z",
  "updated_at": "2026-06-04T14:22:11.482913Z"
}
```

**Hold on to the `id`** — `"8f2a1c4d-9b3e-4a7f-9c2d-1e5b6a7f8c9d"` — you will reference it when you create the deal in Step 3.

**Field reference:**

| Field            | Type    | Notes                                                                 |
|------------------|---------|-----------------------------------------------------------------------|
| `id`             | UUID    | Server-assigned. Stable across renames and merges.                    |
| `platform`       | integer | Auto-resolved from your token.                                        |
| `primary_email`  | string  | Auto-links to a platform user when emails match (case-insensitive).   |
| `platform_user`  | integer | Read-only. Populated once linked to a platform user.                  |
| `active`         | bool    | Read-only. Flips to `false` on user link or merge.                    |
| `lifecycle_stage`| string  | One of `lead`, `qualified`, `opportunity`, `customer`, `churned`.     |
| `metadata`       | object  | Free-form JSON for Platform-defined attributes.                       |

---

### Step 2: Find your default pipeline and stages

Before you can open a deal you need a pipeline `id` and a stage `id`. Filter on `is_default=true` to get the seeded pipeline. The response embeds all six stages inline.

```bash
curl -X GET "https://platform.iblai.app/api/crm/pipelines/?is_default=true" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

**Response — `200 OK`:**

```json
{
  "count": 1,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": 17,
      "platform": 1,
      "name": "Default Pipeline",
      "code": "default",
      "is_default": true,
      "rotten_days": 30,
      "stages": [
        {
          "id": 101,
          "code": "new",
          "name": "New",
          "probability": 10,
          "sort_order": 0,
          "is_won": false,
          "is_lost": false
        },
        {
          "id": 102,
          "code": "qualified",
          "name": "Qualified",
          "probability": 25,
          "sort_order": 1,
          "is_won": false,
          "is_lost": false
        },
        {
          "id": 103,
          "code": "proposal",
          "name": "Proposal",
          "probability": 50,
          "sort_order": 2,
          "is_won": false,
          "is_lost": false
        },
        {
          "id": 104,
          "code": "negotiation",
          "name": "Negotiation",
          "probability": 75,
          "sort_order": 3,
          "is_won": false,
          "is_lost": false
        },
        {
          "id": 105,
          "code": "won",
          "name": "Won",
          "probability": 100,
          "sort_order": 4,
          "is_won": true,
          "is_lost": false
        },
        {
          "id": 106,
          "code": "lost",
          "name": "Lost",
          "probability": 0,
          "sort_order": 5,
          "is_won": false,
          "is_lost": true
        }
      ],
      "metadata": {},
      "created_at": "2026-05-01T09:00:00.000000Z",
      "updated_at": "2026-05-01T09:00:00.000000Z"
    }
  ]
}
```

> **Pagination envelope.** Every list endpoint returns `{count, next_page, previous_page, results}` where `next_page` and `previous_page` are integer page numbers (or `null` at the edges). Pass `?page=2` to walk forward.

> **Reference stages by `code`, not `id`.** Stage `id` values differ between environments (dev vs. staging vs. prod); `code` is stable. Save the pipeline `id` (`17`) for the deal payload below; use `code` everywhere else.

---

### Step 3: Open a deal in the "new" stage

A **Deal** ties a person to a pipeline stage and a monetary value. On create, the `stage` must belong to `pipeline`, both must belong to your Platform, and `status` / `closed_at` are service-managed — the server sets them.

```bash
curl -X POST https://platform.iblai.app/api/crm/deals/ \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Acme — Platform Rollout 2026",
    "description": "Initial conversation about a 200-seat rollout.",
    "person": "8f2a1c4d-9b3e-4a7f-9c2d-1e5b6a7f8c9d",
    "pipeline": 17,
    "stage": 101,
    "lead_value": 48000,
    "currency": "USD",
    "expected_close_date": "2026-09-30"
  }'
```

**Response — `201 Created`:**

```json
{
  "id": 5821,
  "platform": 1,
  "title": "Acme — Platform Rollout 2026",
  "description": "Initial conversation about a 200-seat rollout.",
  "lead_value": "48000.00",
  "currency": "USD",
  "status": "open",
  "lost_reason": "",
  "expected_close_date": "2026-09-30",
  "closed_at": null,
  "person": "8f2a1c4d-9b3e-4a7f-9c2d-1e5b6a7f8c9d",
  "organization": null,
  "pipeline": 17,
  "stage": 101,
  "source": null,
  "owner": 318,
  "tags": [],
  "metadata": {},
  "created_at": "2026-06-04T14:25:03.117204Z",
  "updated_at": "2026-06-04T14:25:03.117204Z"
}
```

The deal opens with `status: "open"` and `closed_at: null`. `owner` defaults to the calling user (here, user id `318`) when you don't supply one.

---

### Step 4: Move the deal forward

Direct writes to `stage` via `PATCH /deals/{id}/` are allowed for repositioning within a pipeline, but the canonical way to transition is `POST /deals/{id}/move-stage/`. The action accepts either `stage_id` or `stage_code` — prefer `stage_code` so payloads stay portable across environments.

```bash
curl -X POST https://platform.iblai.app/api/crm/deals/5821/move-stage/ \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"stage_code": "qualified"}'
```

**Response — `200 OK`:**

```json
{
  "id": 5821,
  "platform": 1,
  "title": "Acme — Platform Rollout 2026",
  "description": "Initial conversation about a 200-seat rollout.",
  "lead_value": "48000.00",
  "currency": "USD",
  "status": "open",
  "lost_reason": "",
  "expected_close_date": "2026-09-30",
  "closed_at": null,
  "person": "8f2a1c4d-9b3e-4a7f-9c2d-1e5b6a7f8c9d",
  "organization": null,
  "pipeline": 17,
  "stage": 102,
  "source": null,
  "owner": 318,
  "tags": [],
  "metadata": {},
  "created_at": "2026-06-04T14:25:03.117204Z",
  "updated_at": "2026-06-04T14:31:47.882901Z"
}
```

> **Side effects of `move-stage/`.** This action does more than swap a foreign key:
> - A **system Activity** is written under this deal: `type="note"`, `title="Stage changed"`, `comment="New → Qualified"`. It will appear in the deal's activity timeline (Section 10).
> - A **`CRM_DEAL_STAGE_CHANGED` notification** is queued for the deal's owner. Wire your in-app inbox to this notification type to surface real-time pipeline movement (Section 12).
> - `Deal.status` is recomputed from the destination stage's `is_won` / `is_lost` flags. Since `qualified` is neither, status stays `"open"`.

---

### Step 5: Close the deal as won

Skip the intermediate stages for the quickstart and jump straight to won. The `won/` action picks the first `is_won=True` stage in the pipeline by `sort_order` (here, the seeded `won` stage at id `105`). Pass `stage_code` only if you have multiple won stages and need a specific one.

```bash
curl -X POST https://platform.iblai.app/api/crm/deals/5821/won/ \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json"
```

**Response — `200 OK`:**

```json
{
  "id": 5821,
  "platform": 1,
  "title": "Acme — Platform Rollout 2026",
  "description": "Initial conversation about a 200-seat rollout.",
  "lead_value": "48000.00",
  "currency": "USD",
  "status": "won",
  "lost_reason": "",
  "expected_close_date": "2026-09-30",
  "closed_at": "2026-06-04T14:36:12.044188Z",
  "person": "8f2a1c4d-9b3e-4a7f-9c2d-1e5b6a7f8c9d",
  "organization": null,
  "pipeline": 17,
  "stage": 105,
  "source": null,
  "owner": 318,
  "tags": [],
  "metadata": {},
  "created_at": "2026-06-04T14:25:03.117204Z",
  "updated_at": "2026-06-04T14:36:12.044188Z"
}
```

The deal is now in the won stage (`105`), `status` is `"won"`, and `closed_at` is stamped to an ISO 8601 datetime. A second system Activity (`Qualified → Won`) is appended to the timeline and a `CRM_DEAL_STAGE_CHANGED` notification fires.

To reopen a deal later, use `move-stage/` to send it back to any non-terminal stage — the server will clear `closed_at` and reset `status` to `"open"` automatically.

---

### What's Next

You have just exercised the spine of the CRM. From here:

- [**Section 8 — Person Onboarding (link / invite / merge)**](#8-person-onboarding-link-invite-merge): bind a person to a platform user, send an invitation, or merge duplicate persons.
- [**Section 9 — Deal Lifecycle**](#9-deal-lifecycle): pipeline and stage administration, the `move-stage/` / `won/` / `lost/` state machine, and the auto-Activity audit trail.
- [**Section 11 — Tagging**](#11-tagging): create tags, attach them to persons / organizations / deals, and filter list endpoints by tag.
- [**Section 12 — Notifications**](#12-notifications): the three CRM notification types, payload shapes, and recipient routing.
- [**Section 7 — API Reference**](#7-api-reference): full endpoint catalog with request / response schemas, filters, and RBAC requirements.

---
## 5. Core Concepts

Before touching an endpoint, lock in the vocabulary. The CRM API leans on a small set of nouns that show up in every URL, payload, filter, and permission code. Get these right once and the rest of the surface reads itself.

| Concept | Definition |
|---|---|
| Platform | The isolation boundary for every CRM record. Determined by the access token on every request — never passed as a query parameter. Two Platforms cannot see each other's persons, organizations, pipelines, deals, activities, or tags. |
| Person | An individual the Platform tracks — a prospect, lead, qualified opportunity, paying customer, or churned account. Carries name, email, phone, lifecycle stage, owner, optional organization, optional linked Platform user, free-form metadata, and tags. Identified by UUID. |
| Organization | A business or institution a person belongs to. Carries `name`, a free-form `address` JSON object, `owner`, `metadata`, and `tags`. Persons reference an organization through a nullable foreign key; deleting an organization nulls the link rather than cascading. Identified by UUID. |
| Pipeline | A named sales process with an ordered set of stages. A Platform may run multiple pipelines (for example "New Business" and "Renewals") and exactly one is flagged `is_default`. Every deal belongs to exactly one pipeline. |
| Stage | A named bucket inside a Pipeline with a probability (0–100), a `sort_order` integer for kanban column placement, and a stable `code` slug safe for client-side switching. Stages flagged `is_won` or `is_lost` are terminal — moving a deal into one stamps `closed_at` and updates `status` automatically. A non-terminal stage with no `is_won`/`is_lost` flag is an in-flight bucket. |
| Lead Source | The channel that produced a deal — the seeded defaults are `web`, `referral`, `cold_call`, and `advertisement`; rename or add your own. Optional foreign key on Deal. Useful for attribution reporting and funnel filtering. |
| Deal | A unit of revenue work — one opportunity moving through a pipeline. Carries title, amount, currency, expected close date, pipeline, current stage, status, owner, primary person, optional organization, optional lead source, metadata, and tags. Deals expose dedicated action endpoints for stage transitions; direct writes to `status` and `closed_at` are rejected. Identified by integer. |
| Activity | A timeline entry attached to either a person or a deal — never both, never neither. Captures calls, meetings, emails, notes, tasks, lunches, and deadlines. Supports scheduling (`scheduled_at`), reminders (`remind_at`), and idempotent completion via `/done/`. Stage transitions on a deal auto-create a `note` activity for audit. Identified by integer. |
| Tag | A reusable label with a name and a 7-character hex color (validated server-side as `^#[0-9A-Fa-f]{6}$`). Names are unique per Platform. Tags attach to persons, organizations, and deals through a uniform attach/detach action convention. Deleting a tag cascades the attachment rows across every host type. Identified by integer. |
| Lifecycle Stage | A coarse-grained classifier on Person describing where the relationship sits — `lead`, `qualified`, `opportunity`, `customer`, or `churned`. Distinct from Pipeline Stage, which describes where a specific deal sits. Drives list filtering and segmentation. Defaults to `lead`. |
| Owner | The Platform user responsible for a record. Set on Person, Organization, and Deal. Drives notification routing (deal-assigned, activity-due, deal-stage-changed all target the owner) and the `owner=<user_id>` list filter used by "my pipeline" and "my accounts" views. Nullable — unowned records are valid and discoverable through `owner__isnull=true`. |
| Metadata | A free-form JSON object available on every resource. The escape hatch for fields that do not warrant a schema change — custom scoring, integration IDs, source-system payloads. Persons and deals additionally support `?metadata__has_key=<fieldName>` list filtering so clients can surface "everyone with a `linkedin_url`" without backend work. |

### Identifier Conventions

URL shape depends on the resource type. People and organizations use UUID strings; everything else uses integers. Plan client-side routing and cache keys accordingly.

| Resource | ID Type | Example URL Tail |
|---|---|---|
| Person | UUID | `/persons/9f1c7e2a-3d4b-4c5e-8f6a-1b2c3d4e5f60/` |
| Organization | UUID | `/organizations/c4a8b1d2-7e3f-4a5b-9c8d-0e1f2a3b4c5d/` |
| Pipeline | Integer | `/pipelines/3/` |
| Stage | Integer | `/pipelines/3/stages/12/` |
| Lead Source | Integer | `/lead-sources/7/` |
| Deal | Integer | `/deals/482/` |
| Activity | Integer | `/activities/9173/` |
| Tag | Integer | `/tags/24/` |

### Vocabulary Tokens

Three closed enumerations show up across filters, payloads, and responses. Memorise them — the API will reject anything outside these sets.

Lifecycle stages on Person (default `lead`):

| Code | Meaning |
|---|---|
| `lead` | New person, not yet qualified. |
| `qualified` | Vetted; matches ideal-customer criteria. |
| `opportunity` | Active sales motion in flight. |
| `customer` | Closed-won and paying. |
| `churned` | Was a customer, now lapsed. |

Deal statuses (server-managed — direct writes to this field are rejected; use the action endpoints):

| Status | How It Gets Set |
|---|---|
| `open` | Initial state on create; persists while the deal sits in non-terminal stages. |
| `won` | Set by `/deals/{id}/won/` or by `/deals/{id}/move-stage/` into a stage flagged `is_won`. `closed_at` is stamped. |
| `lost` | Set by `/deals/{id}/lost/` or by `/deals/{id}/move-stage/` into a stage flagged `is_lost`. `closed_at` is stamped. |

Activity types:

| Type | Typical Use |
|---|---|
| `call` | Phone conversation logged or scheduled. |
| `meeting` | In-person or video meeting. |
| `email` | Outbound or inbound email captured to the timeline. |
| `note` | Free-form text; also the type auto-emitted on deal stage transitions. |
| `task` | Action item with optional `scheduled_at` and `remind_at`. |
| `lunch` | Meal-based meeting, tracked separately for reporting. |
| `deadline` | Time-bounded commitment surfaced on the owner's reminders. |

### Object Graph

The relationships between the core resources, scoped to a single Platform:

```mermaid
erDiagram
    Platform ||--o{ Person : "scopes"
    Platform ||--o{ Organization : "scopes"
    Platform ||--o{ Pipeline : "scopes"
    Platform ||--o{ LeadSource : "scopes"
    Platform ||--o{ Deal : "scopes"
    Platform ||--o{ Activity : "scopes"
    Platform ||--o{ Tag : "scopes"

    Organization ||--o{ Person : "employs"
    Pipeline ||--o{ Stage : "contains"
    Pipeline ||--o{ Deal : "tracks"
    Stage ||--o{ Deal : "currently holds"
    LeadSource ||--o{ Deal : "attributes"

    Person ||--o{ Deal : "primary on"
    Organization ||--o{ Deal : "involved in"

    Person ||--o{ Activity : "timeline of"
    Deal ||--o{ Activity : "timeline of"

    Tag }o--o{ Person : "labels"
    Tag }o--o{ Organization : "labels"
    Tag }o--o{ Deal : "labels"
```

Two structural rules to internalise from the graph:

- An Activity attaches to a person or a deal — never both, never neither. The serializer enforces this at write time.
- Tags are many-to-many against three distinct host types through a single join table; the same tag instance can label a person, an organization, and a deal simultaneously, and counts in each list view reflect that.

---
## 6. Resource Reference

The CRM exposes eight resources under `/api/crm/`. Each is Platform-scoped (resolved from the caller's token) except `Stage`, which nests under a parent `Pipeline`. IDs are integers for everything except `Person` and `Organization`, which use UUID strings so they can be minted client-side and survive cross-system imports. This section is the at-a-glance map; Section 7 walks every endpoint, payload, and filter in detail.

| Resource | URL Prefix | ID Type | Scoped To | Description |
|---|---|---|---|---|
| Person | `/api/crm/persons/` | UUID string | Platform | A human record, optionally linked to a Platform user |
| Organization | `/api/crm/organizations/` | UUID string | Platform | A business a person belongs to |
| Pipeline | `/api/crm/pipelines/` | integer | Platform | An ordered sequence of stages a deal flows through |
| Stage | `/api/crm/pipelines/{pipeline_id}/stages/` | integer | Pipeline | A named bucket inside a pipeline |
| Lead Source | `/api/crm/lead-sources/` | integer | Platform | Where a deal originated (web, referral, …) |
| Deal | `/api/crm/deals/` | integer | Platform | A revenue opportunity attached to a person |
| Activity | `/api/crm/activities/` | integer | Platform | A logged or scheduled interaction |
| Tag | `/api/crm/tags/` | integer | Platform | A coloured label attachable to people, organizations, and deals |

### Read & Write Shape

Every resource returns three standard envelope fields on read:

- `created_at` — ISO-8601 timestamp, set on insert
- `updated_at` — ISO-8601 timestamp, refreshed on every save
- `metadata` — free-form JSON object the integrator owns end-to-end (see Section 5 on the metadata escape hatch)

A subset additionally carries an `owner` field — a foreign key to the Platform user accountable for the record. `owner` is present on `Person`, `Organization`, `Deal`, and `Activity`. It is writable on create/update and used by the `owner` list filter described in Section 7.

`Person`, `Organization`, and `Deal` also expose a read-only `tags` array on detail and list reads. The array is a denormalised projection of attached `Tag` records; you mutate it through the dedicated attach/detach endpoints documented in Section 11, never by PATCHing the host resource.

### Server-Managed Fields

A handful of fields are reserved by the server. Clients must not send them on create or update — the API will either ignore them silently or reject the payload depending on the serializer. They appear on reads so the UI can render derived state without recomputing it.

| Resource | Fields | Why |
|---|---|---|
| Person | `platform_user`, `active` | Set when linked to a Platform user (Section 8) |
| Deal | `status`, `closed_at` | Derived from stage + won/lost actions (Section 9) |
| Activity | `done_at`, `reminder_sent` | Stamped automatically on completion |

`Person.platform_user` is populated by the `/link-user/` action (or by the auto-link signal when a Platform user with a matching email is created); `Person.active` flips to `false` the moment a Platform user binding lands, signalling "this person is now a real authenticated user — stop treating them as a lead". Both fields are read-only thereafter.

`Deal.status` is derived from the current stage's terminal flag plus any `/won/` or `/lost/` action; `closed_at` is stamped the moment status leaves `open`. `Activity.done_at` is stamped by the idempotent `/done/` action, and `reminder_sent` is set by the reminder dispatcher when the scheduled reminder fires. See Sections 9 and 11 for the full state machines.

---
## 7. API Reference

**Base path:** `/api/crm/`

**Authentication:** `Token` header required on every endpoint. Pass it as `Authorization: Token YOUR_ACCESS_TOKEN`. The Platform is inferred from the token — do not pass `platform_key` on the URL.

**Pagination:** Every list endpoint returns the envelope `{count, next_page, previous_page, results}` with integer page numbers. Use `?page=N&page_size=N` (max `page_size` = 100).

**IDs:** Persons and Organizations use UUID primary keys. Pipelines, PipelineStages, LeadSources, Deals, Activities, Tags and tag assignments use integer primary keys.

---

### 7.1 Persons

#### Summary

| Method | Endpoint | Auth | RBAC Permission | Description |
|--------|----------|------|-----------------|-------------|
| GET    | `/persons/`                            | Token | `Ibl.CRM/Persons/list`   | Paginated list of persons in your Platform |
| POST   | `/persons/`                            | Token | `Ibl.CRM/Persons/action` | Create a person |
| GET    | `/persons/{id}/`                       | Token | `Ibl.CRM/Persons/read`   | Retrieve one person |
| PUT    | `/persons/{id}/`                       | Token | `Ibl.CRM/Persons/write`  | Replace all editable fields |
| PATCH  | `/persons/{id}/`                       | Token | `Ibl.CRM/Persons/write`  | Update a subset of fields |
| DELETE | `/persons/{id}/`                       | Token | `Ibl.CRM/Persons/delete` | Delete a person |
| POST   | `/persons/{id}/link-user/`             | Token | `Ibl.CRM/Persons/write`  | Bind a person to an existing Platform user |
| POST   | `/persons/{id}/invite/`                | Token | `Ibl.CRM/Invite/action`  | Send a Platform invitation to the person's primary email |
| POST   | `/persons/merge/`                      | Token | `Ibl.CRM/Persons/write`  | Merge duplicate persons into a primary |
| POST   | `/persons/{id}/tags/`                  | Token | `Ibl.CRM/Tags/write`     | Attach a Tag to the person |
| DELETE | `/persons/{id}/tags/{tag_id}/`         | Token | `Ibl.CRM/Tags/write`     | Detach a Tag from the person |

---

#### GET `/persons/`

List persons in your Platform. Paginated.

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `lifecycle_stage` | string | No | Filter by funnel position: `lead`, `qualified`, `opportunity`, `customer`, `churned` |
| `owner` | integer | No | Filter by owning Platform user id |
| `organization` | UUID | No | Filter by organization id |
| `created_at__gte` | ISO 8601 | No | Persons created on or after this timestamp |
| `created_at__lte` | ISO 8601 | No | Persons created on or before this timestamp |
| `metadata__has_key` | string | No | Return only persons whose `metadata` JSON has the given top-level key |
| `tags` | integer (repeatable) | No | Filter by attached Tag id. OR semantics: `?tags=1&tags=2` or `?tags=1,2` returns persons carrying at least one of the tags. Results are de-duplicated |
| `page` | integer | No | Page number |
| `page_size` | integer | No | Rows per page (max 100) |

**Response** `200 OK`

```json
{
  "count": 2,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "platform": 1,
      "name": "Alice Chen",
      "primary_email": "alice@example.com",
      "emails": [
        {"label": "work", "email": "alice@example.com"}
      ],
      "contact_numbers": [
        {"label": "mobile", "number": "+15551234567"}
      ],
      "job_title": "VP Engineering",
      "organization": "8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111",
      "owner": 42,
      "platform_user": null,
      "lifecycle_stage": "lead",
      "unique_id": "",
      "active": true,
      "tags": [
        {"id": 7, "name": "VIP", "color": "#ff5722"}
      ],
      "metadata": {"source_campaign": "q2-webinar"},
      "created_at": "2026-05-12T09:14:01.123456Z",
      "updated_at": "2026-05-12T09:14:01.123456Z"
    },
    {
      "id": "7cb93e21-1234-4abc-9876-def012345678",
      "platform": 1,
      "name": "Bob Singh",
      "primary_email": "bob@example.com",
      "emails": [],
      "contact_numbers": [],
      "job_title": "",
      "organization": null,
      "owner": null,
      "platform_user": 1184,
      "lifecycle_stage": "customer",
      "unique_id": "salesforce-00Q3X000001abcd",
      "active": false,
      "tags": [],
      "metadata": {},
      "created_at": "2026-04-30T11:02:55.000000Z",
      "updated_at": "2026-05-12T10:00:00.000000Z"
    }
  ]
}
```

**Response fields**

| Field | Type | Description |
|-------|------|-------------|
| `count` | integer | Total persons matching the filters |
| `next_page` | integer or null | Next page number (null when on last page) |
| `previous_page` | integer or null | Previous page number (null when on first page) |
| `results[].id` | UUID | Server-assigned. Stable across renames and merges |
| `results[].platform` | string | Platform key. Set server-side from the auth token |
| `results[].name` | string | Display name |
| `results[].primary_email` | string or null | Canonical email used by the auto-link signal (case-insensitive match against new Platform users) |
| `results[].emails` | array of objects | Additional emails. Free-form shape; the recommended pair is `{label, email}` |
| `results[].contact_numbers` | array of objects | Phone numbers. Free-form shape; the recommended pair is `{label, number}` |
| `results[].job_title` | string | Display only; not validated |
| `results[].organization` | UUID or null | Organization in your Platform |
| `results[].owner` | integer or null | Platform user id of the internal account manager |
| `results[].platform_user` | integer or null | Read-only. Populated once this person is bound to a Platform user |
| `results[].lifecycle_stage` | string | One of `lead`, `qualified`, `opportunity`, `customer`, `churned` |
| `results[].unique_id` | string | External system id (e.g. import key from another CRM). Unique per Platform when non-blank; up to 128 chars |
| `results[].active` | boolean | Read-only. Flipped to `false` when bound to a Platform user or merged into another person |
| `results[].tags` | array of objects | Flat `{id, name, color}` per attached Tag. Always present; empty array when none |
| `results[].metadata` | object | Free-form JSON for Platform-defined attributes |
| `results[].created_at` | ISO 8601 | Creation timestamp |
| `results[].updated_at` | ISO 8601 | Last-modified timestamp |

```bash
curl -X GET "https://platform.iblai.app/api/crm/persons/?lifecycle_stage=qualified&page=1&page_size=25" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/persons/`

Create a person. The Platform is set server-side from your credentials.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Display name |
| `primary_email` | string | No | Canonical email. When a Platform user is later created with a matching email (case-insensitive), this person is auto-linked |
| `emails` | array of objects | No | Additional emails (defaults to `[]`) |
| `contact_numbers` | array of objects | No | Phone numbers (defaults to `[]`) |
| `job_title` | string | No | Display only (defaults to `""`) |
| `organization` | UUID | No | Organization id in your Platform. References to another Platform's organization are rejected |
| `owner` | integer | No | Id of the owning Platform user. Must be an active member of your Platform |
| `lifecycle_stage` | string | No | Defaults to `lead`. One of `lead`, `qualified`, `opportunity`, `customer`, `churned` |
| `unique_id` | string | No | External import key. Up to 128 chars; unique per Platform when non-blank |
| `metadata` | object | No | Free-form JSON (defaults to `{}`) |

```json
{
  "name": "Alice Chen",
  "primary_email": "alice@example.com",
  "emails": [
    {"label": "work", "email": "alice@example.com"}
  ],
  "contact_numbers": [
    {"label": "mobile", "number": "+15551234567"}
  ],
  "job_title": "VP Engineering",
  "lifecycle_stage": "lead",
  "metadata": {"source_campaign": "q2-webinar"}
}
```

**Response** `201 Created` — full person object (same shape as the list `results[]` entry above).

**Error** `400 Bad Request` — validation error (duplicate `unique_id`, cross-Platform `organization`, owner not in Platform, etc.)

```json
{
  "unique_id": [
    "A Person with unique_id 'salesforce-00Q3X000001abcd' already exists on this Platform."
  ]
}
```

**Error** `403 Forbidden` — caller is missing `Ibl.CRM/Persons/action`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/persons/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Chen",
    "primary_email": "alice@example.com",
    "job_title": "VP Engineering",
    "lifecycle_stage": "lead",
    "metadata": {"source_campaign": "q2-webinar"}
  }'
```

---

#### GET `/persons/{id}/`

Retrieve a single person.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | UUID | Person id |

**Response** `200 OK` — full person object (same shape as the list `results[]` entry above).

**Error** `404 Not Found` — person does not exist, or belongs to a different Platform (existence is not leaked).

```bash
curl -X GET "https://platform.iblai.app/api/crm/persons/3fa85f64-5717-4562-b3fc-2c963f66afa6/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### PUT `/persons/{id}/`

Replace every editable field on the person. Fields you omit are reset to their serializer defaults.

**Request body** — same fields as `POST /persons/` (server-managed fields `id`, `platform`, `platform_user`, `active`, `created_at`, `updated_at` are ignored if sent).

**Response** `200 OK` — full updated person object.

**Error** `400` — validation error. **Error** `404` — not found.

```bash
curl -X PUT "https://platform.iblai.app/api/crm/persons/3fa85f64-5717-4562-b3fc-2c963f66afa6/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Chen",
    "primary_email": "alice@acme.com",
    "lifecycle_stage": "qualified",
    "job_title": "VP of Engineering",
    "organization": "8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111",
    "owner": 42,
    "metadata": {}
  }'
```

---

#### PATCH `/persons/{id}/`

Update a subset of fields. Only fields present in the body are touched.

```json
{
  "job_title": "VP of Engineering",
  "lifecycle_stage": "qualified"
}
```

**Response** `200 OK` — full updated person object.

```bash
curl -X PATCH "https://platform.iblai.app/api/crm/persons/3fa85f64-5717-4562-b3fc-2c963f66afa6/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"job_title": "VP of Engineering", "lifecycle_stage": "qualified"}'
```

---

#### DELETE `/persons/{id}/`

Hard-delete the person.

**Response** `204 No Content`.

**Error** `404 Not Found`.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/persons/3fa85f64-5717-4562-b3fc-2c963f66afa6/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/persons/{id}/link-user/`

Bind a person to an existing Platform user. The target user must already be an active member of your Platform — if they are not, send an invitation via `POST /persons/{id}/invite/` instead.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | UUID | Person id |

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `user_id` | integer | Yes | Id of the Platform user to bind to this person |

```json
{"user_id": 1184}
```

**Response** `200 OK` — the full person object. On a successful bind, `platform_user` is set to the requested `user_id` and `active` flips to `false`.

```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "platform": 1,
  "name": "Alice Chen",
  "primary_email": "alice@example.com",
  "emails": [],
  "contact_numbers": [],
  "job_title": "VP Engineering",
  "organization": null,
  "owner": 42,
  "platform_user": 1184,
  "lifecycle_stage": "qualified",
  "unique_id": "",
  "active": false,
  "tags": [],
  "metadata": {},
  "created_at": "2026-05-12T09:14:01.123456Z",
  "updated_at": "2026-06-04T08:00:00.000000Z"
}
```

**Silent-refusal footgun.** A 200 response whose `platform_user` does **not** equal the `user_id` you sent means the person was already bound to a different Platform user and the existing binding was preserved — the API will not silently re-parent a person from one user to another. Always assert `response.platform_user == request.user_id` before treating the call as a successful bind.

**Status codes**

| Code | When |
|------|------|
| `200` | Success, OR silent refusal (person already bound to a different user) |
| `400` | `user_id` missing or not an integer |
| `403` | Caller missing `Ibl.CRM/Persons/write`, OR target user has no active `UserPlatformLink` to your Platform |
| `404` | Person or user does not exist |

**Error** `403` — target user is not a Platform member.

```json
{
  "detail": "User has no active UserPlatformLink to this platform; issue an invitation instead."
}
```

**Error** `404` — user does not exist.

```json
{"detail": "User not found."}
```

```bash
curl -X POST "https://platform.iblai.app/api/crm/persons/3fa85f64-5717-4562-b3fc-2c963f66afa6/link-user/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1184}'
```

---

#### POST `/persons/{id}/invite/`

Send a Platform invitation to the person's `primary_email`. On acceptance, the invitee joins your Platform with the privileges configured here and the person is auto-linked to the new user account.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | UUID | Person id |

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `is_admin` | boolean | No | Grant Platform-admin privileges on acceptance. Defaults to `false` |
| `is_staff` | boolean | No | Grant staff privileges on acceptance. Defaults to `false` |
| `enrollment_config` | object | No | Auto-enrollment payload forwarded to the Platform invitation (courses, programs, pathways) |
| `redirect_to` | string | No | URL the invitee is sent to after accepting (e.g. a custom landing page). Max 255 chars. Omit to use the Platform default |

```json
{
  "is_admin": false,
  "is_staff": false,
  "redirect_to": "https://acme.iblai.app/welcome"
}
```

**Response** `201 Created`

```json
{
  "person_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "invitation_id": 8821,
  "invitation_email": "alice@example.com",
  "platform_key": "acme-learning",
  "auto_accept": true,
  "active": true,
  "redirect_to": "https://acme.iblai.app/welcome",
  "created": "2026-06-04T08:12:01.000000+00:00"
}
```

**Response fields**

| Field | Type | Description |
|-------|------|-------------|
| `person_id` | UUID | The invited person's id |
| `invitation_id` | integer | Id of the newly created Platform invitation |
| `invitation_email` | string | Email the invitation was sent to (the person's `primary_email`) |
| `platform_key` | string | Platform the invitation targets |
| `auto_accept` | boolean | Whether the invitation is auto-accepted when the invitee first signs in (always `true` for CRM invitations) |
| `active` | boolean | Whether the invitation is currently active |
| `redirect_to` | string or null | Post-acceptance destination |
| `created` | ISO 8601 or null | When the invitation was created |

**Status codes**

| Code | When |
|------|------|
| `201` | Invitation sent |
| `400` | Person has no `primary_email` on record |
| `403` | Caller missing `Ibl.CRM/Invite/action` |
| `404` | Person does not exist |
| `409` | An active Platform invitation already exists for this email + Platform |
| `422` | Person is already linked to a Platform user |

**Error** `400` — no email on record.

```json
{"detail": "Person has no primary_email; cannot invite."}
```

**Error** `409 Conflict` — pending invitation exists. The body carries the pre-existing invitation id so you can dedup or surface a "resend" UI.

```json
{
  "detail": "Active PlatformInvitation already exists for this email + platform.",
  "invitation_id": 8519,
  "person_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "platform_key": "acme-learning"
}
```

**Error** `422 Unprocessable Entity` — person already bound to a Platform user.

```json
{"detail": "Person already linked to a platform user."}
```

```bash
curl -X POST "https://platform.iblai.app/api/crm/persons/3fa85f64-5717-4562-b3fc-2c963f66afa6/invite/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "is_admin": false,
    "is_staff": false,
    "redirect_to": "https://acme.iblai.app/welcome"
  }'
```

---

#### POST `/persons/merge/`

Merge duplicate persons into a primary. Inside a single transaction every Deal and Activity pointing at a duplicate is re-parented onto the primary, tag assignments are moved (or dropped silently when the primary already carries that tag — the `(tag, person)` unique constraint forbids stacking), and each duplicate is soft-deleted (`active=false`, row still retrievable by id).

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `primary_id` | UUID | Yes | The surviving person. Must belong to your Platform |
| `duplicate_ids` | array of UUIDs | Yes | Persons to merge into the primary. Must be non-empty, must all belong to your Platform, and must not include `primary_id` |

```json
{
  "primary_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "duplicate_ids": [
    "7cb93e21-1234-4abc-9876-def012345678",
    "9d2a4f08-aaaa-4bbb-8ccc-1111d2e3f444"
  ]
}
```

**Response** `200 OK`

```json
{
  "primary_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "merged_ids": [
    "7cb93e21-1234-4abc-9876-def012345678",
    "9d2a4f08-aaaa-4bbb-8ccc-1111d2e3f444"
  ],
  "reparented": {
    "deals": 4,
    "activities": 11,
    "tags": 3
  }
}
```

**Response fields**

| Field | Type | Description |
|-------|------|-------------|
| `primary_id` | UUID | Surviving person id |
| `merged_ids` | array of UUIDs | Persons that were marked inactive by this call |
| `reparented.deals` | integer | Deals re-parented onto the primary |
| `reparented.activities` | integer | Activities re-parented onto the primary |
| `reparented.tags` | integer | Tag assignments *touched* (moved + dropped). A duplicate that shared a tag with the primary still counts here even though its row was deleted instead of moved |

**Status codes**

| Code | When |
|------|------|
| `200` | Merge complete (also returned on re-running with all-already-merged duplicates — counts are zero) |
| `400` | `primary_id` appears in `duplicate_ids`, `duplicate_ids` is empty, or one or more duplicates belong to another Platform |
| `403` | Caller missing `Ibl.CRM/Persons/write` |
| `404` | Primary person not found in your Platform |

**Error** `400` — primary in duplicates.

```json
{"detail": "primary_id may not appear in duplicate_ids."}
```

**Error** `400` — cross-Platform duplicate.

```json
{"detail": "All persons must belong to the same platform."}
```

**Error** `404` — primary missing.

```json
{"detail": "Primary Person not found in this platform."}
```

```bash
curl -X POST "https://platform.iblai.app/api/crm/persons/merge/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "primary_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "duplicate_ids": ["7cb93e21-1234-4abc-9876-def012345678"]
  }'
```

---

#### POST `/persons/{id}/tags/`

Attach an existing Tag to the person. The Tag must belong to your Platform.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tag_id` | integer | Yes | Id of a Tag in your Platform |

```json
{"tag_id": 7}
```

**Response** `201 Created`

```json
{
  "assignment_id": 312,
  "tag": {"id": 7, "name": "VIP", "color": "#ff5722"}
}
```

**Error** `409 Conflict` — tag is already attached. The existing assignment id is returned so the client can keep its local state consistent without a re-fetch.

```json
{
  "detail": "Tag already attached.",
  "assignment_id": 312
}
```

**Error** `404 Not Found` — the tag does not exist or belongs to another Platform (existence is not leaked).

```bash
curl -X POST "https://platform.iblai.app/api/crm/persons/3fa85f64-5717-4562-b3fc-2c963f66afa6/tags/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tag_id": 7}'
```

---

#### DELETE `/persons/{id}/tags/{tag_id}/`

Detach a Tag from the person.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | UUID | Person id |
| `tag_id` | integer | Tag id |

**Response** `204 No Content`.

**Error** `404 Not Found` — the tag was not attached to this person.

```json
{"detail": "Tag not attached to this record."}
```

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/persons/3fa85f64-5717-4562-b3fc-2c963f66afa6/tags/7/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

### 7.2 Organizations

#### Summary

| Method | Endpoint | Auth | RBAC Permission | Description |
|--------|----------|------|-----------------|-------------|
| GET    | `/organizations/`                            | Token | `Ibl.CRM/Organizations/list`   | Paginated list of organizations in your Platform |
| POST   | `/organizations/`                            | Token | `Ibl.CRM/Organizations/action` | Create an organization |
| GET    | `/organizations/{id}/`                       | Token | `Ibl.CRM/Organizations/read`   | Retrieve one organization |
| PUT    | `/organizations/{id}/`                       | Token | `Ibl.CRM/Organizations/write`  | Replace all editable fields |
| PATCH  | `/organizations/{id}/`                       | Token | `Ibl.CRM/Organizations/write`  | Update a subset of fields |
| DELETE | `/organizations/{id}/`                       | Token | `Ibl.CRM/Organizations/delete` | Delete an organization |
| POST   | `/organizations/{id}/tags/`                  | Token | `Ibl.CRM/Tags/write`           | Attach a Tag to the organization |
| DELETE | `/organizations/{id}/tags/{tag_id}/`         | Token | `Ibl.CRM/Tags/write`           | Detach a Tag from the organization |

---

#### GET `/organizations/`

List organizations in your Platform. Paginated.

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `owner` | integer | No | Filter by owning Platform user id |
| `name` | string | No | Case-insensitive substring match on the organization name |
| `tags` | integer (repeatable) | No | Filter by attached Tag id. OR semantics; results de-duplicated |
| `page` | integer | No | Page number |
| `page_size` | integer | No | Rows per page (max 100) |

**Response** `200 OK`

```json
{
  "count": 1,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": "8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111",
      "platform": 1,
      "name": "Acme Inc",
      "address": {
        "street": "123 Main St",
        "city": "New York",
        "state": "NY",
        "zip": "10001",
        "country": "US"
      },
      "owner": 42,
      "tags": [
        {"id": 11, "name": "Strategic Account", "color": "#1976d2"}
      ],
      "metadata": {"industry": "edtech"},
      "created_at": "2026-05-12T09:00:00.000000Z",
      "updated_at": "2026-05-20T14:33:21.000000Z"
    }
  ]
}
```

**Response fields**

| Field | Type | Description |
|-------|------|-------------|
| `count` | integer | Total organizations matching the filters |
| `next_page` / `previous_page` | integer or null | Pagination cursors |
| `results[].id` | UUID | Server-assigned |
| `results[].platform` | string | Platform key. Set server-side from the auth token |
| `results[].name` | string | Display name. Unique per Platform, case-sensitive, trimmed of surrounding whitespace before comparison. Max 255 chars |
| `results[].address` | object | Free-form JSON. The recommended (but not enforced) shape is `{street, city, state, zip, country}` |
| `results[].owner` | integer or null | Platform user id of the internal account manager. Must be an active member of your Platform |
| `results[].tags` | array of objects | Flat `{id, name, color}` per attached Tag |
| `results[].metadata` | object | Free-form JSON |
| `results[].created_at` | ISO 8601 | Creation timestamp |
| `results[].updated_at` | ISO 8601 | Last-modified timestamp |

```bash
curl -X GET "https://platform.iblai.app/api/crm/organizations/?name=acme" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/organizations/`

Create an organization.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Display name. Must be unique within your Platform (case-sensitive, trimmed). Max 255 chars |
| `address` | object | No | Free-form JSON (defaults to `{}`) |
| `owner` | integer | No | Owning Platform user id |
| `metadata` | object | No | Free-form JSON (defaults to `{}`) |

```json
{
  "name": "Acme Inc",
  "address": {
    "street": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zip": "10001",
    "country": "US"
  },
  "metadata": {"industry": "edtech"}
}
```

**Response** `201 Created` — full organization object.

**Error** `400 Bad Request` — duplicate name.

```json
{
  "name": [
    "An Organization with name 'Acme Inc' already exists on this Platform."
  ]
}
```

**Error** `400 Bad Request` — blank name.

```json
{"name": ["Name must not be blank."]}
```

**Error** `403 Forbidden` — caller missing `Ibl.CRM/Organizations/action`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/organizations/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Inc",
    "address": {"street": "123 Main St", "city": "New York", "state": "NY", "zip": "10001", "country": "US"},
    "metadata": {"industry": "edtech"}
  }'
```

---

#### GET `/organizations/{id}/`

Retrieve an organization.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `id` | UUID | Organization id |

**Response** `200 OK` — full organization object.

**Error** `404 Not Found` — organization does not exist or belongs to another Platform.

```bash
curl -X GET "https://platform.iblai.app/api/crm/organizations/8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### PUT `/organizations/{id}/`

Replace every editable field on the organization.

**Request body** — same fields as `POST /organizations/`.

**Response** `200 OK` — full updated organization object.

```bash
curl -X PUT "https://platform.iblai.app/api/crm/organizations/8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Corporation",
    "address": {"city": "Brooklyn"},
    "owner": 42,
    "metadata": {"industry": "edtech"}
  }'
```

---

#### PATCH `/organizations/{id}/`

Update a subset of fields.

```json
{
  "name": "Acme Corporation",
  "address": {"city": "Brooklyn"}
}
```

**Response** `200 OK` — full updated organization object.

```bash
curl -X PATCH "https://platform.iblai.app/api/crm/organizations/8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme Corporation"}'
```

---

#### DELETE `/organizations/{id}/`

Hard-delete the organization. Persons and Deals that referenced it are kept; their `organization` foreign key is set to `null` (no cascade-delete of people or deals). Attached tag assignments are removed.

**Response** `204 No Content`.

**Error** `404 Not Found`.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/organizations/8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/organizations/{id}/tags/`

Attach an existing Tag to the organization.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tag_id` | integer | Yes | Id of a Tag in your Platform |

```json
{"tag_id": 11}
```

**Response** `201 Created`

```json
{
  "assignment_id": 88,
  "tag": {"id": 11, "name": "Strategic Account", "color": "#1976d2"}
}
```

**Error** `409 Conflict` — already attached.

```json
{
  "detail": "Tag already attached.",
  "assignment_id": 88
}
```

**Error** `404 Not Found` — tag belongs to another Platform.

```bash
curl -X POST "https://platform.iblai.app/api/crm/organizations/8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111/tags/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tag_id": 11}'
```

---

#### DELETE `/organizations/{id}/tags/{tag_id}/`

Detach a Tag from the organization.

**Response** `204 No Content`.

**Error** `404 Not Found` — the tag was not attached.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/organizations/8c6a1b22-5b7e-4d9d-9b41-21c2e6e4f111/tags/11/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

### 7.3 Pipelines & Stages

Pipelines are top-level resources at `/pipelines/`. Stages are **nested** under their parent pipeline at `/pipelines/{pipeline_id}/stages/` — a stage's `pipeline` is immutable, you cannot move a stage between pipelines, and the nested route refuses access to stages of pipelines on other Platforms.

#### Summary

| Method | Endpoint | Auth | RBAC Permission | Description |
|--------|----------|------|-----------------|-------------|
| GET    | `/pipelines/`                                            | Token | `Ibl.CRM/Pipelines/list`   | Paginated list of pipelines, each with its ordered stages inline |
| POST   | `/pipelines/`                                            | Token | `Ibl.CRM/Pipelines/action` | Create a pipeline |
| GET    | `/pipelines/{id}/`                                       | Token | `Ibl.CRM/Pipelines/read`   | Retrieve one pipeline |
| PUT    | `/pipelines/{id}/`                                       | Token | `Ibl.CRM/Pipelines/write`  | Replace all editable fields |
| PATCH  | `/pipelines/{id}/`                                       | Token | `Ibl.CRM/Pipelines/write`  | Update a subset of fields |
| DELETE | `/pipelines/{id}/`                                       | Token | `Ibl.CRM/Pipelines/delete` | Delete a pipeline (cascades to its stages) |
| GET    | `/pipelines/{pipeline_id}/stages/`                       | Token | `Ibl.CRM/Pipelines/list`   | List stages for a pipeline |
| POST   | `/pipelines/{pipeline_id}/stages/`                       | Token | `Ibl.CRM/Pipelines/action` | Create a stage in this pipeline |
| GET    | `/pipelines/{pipeline_id}/stages/{id}/`                  | Token | `Ibl.CRM/Pipelines/read`   | Retrieve one stage |
| PUT    | `/pipelines/{pipeline_id}/stages/{id}/`                  | Token | `Ibl.CRM/Pipelines/write`  | Replace all editable fields |
| PATCH  | `/pipelines/{pipeline_id}/stages/{id}/`                  | Token | `Ibl.CRM/Pipelines/write`  | Update a subset of fields |
| DELETE | `/pipelines/{pipeline_id}/stages/{id}/`                  | Token | `Ibl.CRM/Pipelines/delete` | Delete a stage |

**Seeded defaults.** Every newly created Platform is seeded with one pipeline (`code=default`, `name=Default Pipeline`, `is_default=true`, `rotten_days=30`) carrying six stages — `new`, `qualified`, `proposal`, `negotiation`, `won` (terminal `is_won`), and `lost` (terminal `is_lost`). Existing Platforms were backfilled by a data migration.

---

#### GET `/pipelines/`

List pipelines in your Platform. Each pipeline carries its ordered `stages` inline.

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `code` | string | No | Case-insensitive exact match on `code` |
| `name` | string | No | Case-insensitive substring match on `name` |
| `is_default` | boolean | No | Filter to the default pipeline (or non-defaults) |
| `page` | integer | No | Page number |
| `page_size` | integer | No | Rows per page (max 100) |

**Response** `200 OK`

```json
{
  "count": 1,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": 1,
      "platform": 1,
      "name": "Default Pipeline",
      "code": "default",
      "is_default": true,
      "rotten_days": 30,
      "stages": [
        {"id": 1, "pipeline": 1, "code": "new",         "name": "New",         "probability": 10,  "sort_order": 0, "is_won": false, "is_lost": false, "metadata": {}, "created_at": "2026-01-15T00:00:00Z", "updated_at": "2026-01-15T00:00:00Z"},
        {"id": 2, "pipeline": 1, "code": "qualified",   "name": "Qualified",   "probability": 25,  "sort_order": 1, "is_won": false, "is_lost": false, "metadata": {}, "created_at": "2026-01-15T00:00:00Z", "updated_at": "2026-01-15T00:00:00Z"},
        {"id": 3, "pipeline": 1, "code": "proposal",    "name": "Proposal",    "probability": 50,  "sort_order": 2, "is_won": false, "is_lost": false, "metadata": {}, "created_at": "2026-01-15T00:00:00Z", "updated_at": "2026-01-15T00:00:00Z"},
        {"id": 4, "pipeline": 1, "code": "negotiation", "name": "Negotiation", "probability": 75,  "sort_order": 3, "is_won": false, "is_lost": false, "metadata": {}, "created_at": "2026-01-15T00:00:00Z", "updated_at": "2026-01-15T00:00:00Z"},
        {"id": 5, "pipeline": 1, "code": "won",         "name": "Won",         "probability": 100, "sort_order": 4, "is_won": true,  "is_lost": false, "metadata": {}, "created_at": "2026-01-15T00:00:00Z", "updated_at": "2026-01-15T00:00:00Z"},
        {"id": 6, "pipeline": 1, "code": "lost",        "name": "Lost",        "probability": 0,   "sort_order": 5, "is_won": false, "is_lost": true,  "metadata": {}, "created_at": "2026-01-15T00:00:00Z", "updated_at": "2026-01-15T00:00:00Z"}
      ],
      "metadata": {},
      "created_at": "2026-01-15T00:00:00Z",
      "updated_at": "2026-01-15T00:00:00Z"
    }
  ]
}
```

**Response fields**

| Field | Type | Description |
|-------|------|-------------|
| `results[].id` | integer | Server-assigned |
| `results[].platform` | string | Platform key. Set server-side from the auth token |
| `results[].name` | string | Display name (e.g. `"B2B Sales"`, `"Renewals"`) |
| `results[].code` | string | Stable slug used in API payloads. Unique per Platform; lowercase letters, digits, hyphens |
| `results[].is_default` | boolean | At most one default per Platform. New Deals land here when no pipeline is specified |
| `results[].rotten_days` | integer | Days a Deal can sit in a single stage before it's surfaced as stale (defaults to 30) |
| `results[].stages` | array of objects | Read-only inline list of stages, ordered by `sort_order` |
| `results[].metadata` | object | Free-form JSON |
| `results[].created_at` / `updated_at` | ISO 8601 | Timestamps |

```bash
curl -X GET "https://platform.iblai.app/api/crm/pipelines/?is_default=true" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/pipelines/`

Create a pipeline.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Display name |
| `code` | string | Yes | Stable slug. Unique per Platform; lowercase letters, digits, hyphens |
| `is_default` | boolean | No | Defaults to `false`. At most one default per Platform — un-flag the existing default first if you want to promote a new one |
| `rotten_days` | integer | No | Defaults to `30` |
| `metadata` | object | No | Free-form JSON (defaults to `{}`) |

```json
{
  "name": "B2B Sales",
  "code": "b2b-sales",
  "is_default": false,
  "rotten_days": 45,
  "metadata": {"team": "enterprise"}
}
```

**Response** `201 Created` — full pipeline object. The `stages` array is initially empty; add stages via `POST /pipelines/{id}/stages/`.

**Error** `400 Bad Request` — duplicate code.

```json
{
  "code": [
    "A Pipeline with code 'b2b-sales' already exists on this Platform."
  ]
}
```

**Error** `400 Bad Request` — a default already exists.

```json
{
  "is_default": [
    "Another Pipeline on this Platform is already the default. Un-flag the existing default first."
  ]
}
```

```bash
curl -X POST "https://platform.iblai.app/api/crm/pipelines/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "B2B Sales",
    "code": "b2b-sales",
    "is_default": false,
    "rotten_days": 45,
    "metadata": {"team": "enterprise"}
  }'
```

---

#### GET `/pipelines/{id}/`

Retrieve one pipeline, including its ordered `stages`.

**Response** `200 OK` — full pipeline object (same shape as the list `results[]` entry).

```bash
curl -X GET "https://platform.iblai.app/api/crm/pipelines/1/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### PUT `/pipelines/{id}/` and PATCH `/pipelines/{id}/`

Replace or update. `stages` is read-only on the pipeline — manage stages through the nested endpoints.

```json
{
  "name": "B2B Sales — Enterprise",
  "rotten_days": 60
}
```

**Response** `200 OK` — full updated pipeline object.

```bash
curl -X PATCH "https://platform.iblai.app/api/crm/pipelines/1/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "B2B Sales — Enterprise", "rotten_days": 60}'
```

---

#### DELETE `/pipelines/{id}/`

Delete a pipeline. The pipeline's stages cascade-delete with it.

**Response** `204 No Content`.

**Error** `409 Conflict` — at least one Deal still references this pipeline. Move or delete those deals first.

```json
{"detail": "Pipeline still has Deals attached."}
```

**Error** `404 Not Found`.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/pipelines/1/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### GET `/pipelines/{pipeline_id}/stages/`

List stages for a pipeline, ordered by `sort_order`. Paginated.

**Path parameters**

| Name | Type | Description |
|------|------|-------------|
| `pipeline_id` | integer | Pipeline id |

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `code` | string | No | Case-insensitive exact match |
| `is_won` | boolean | No | Filter to terminal won stages |
| `is_lost` | boolean | No | Filter to terminal lost stages |
| `page` / `page_size` | integer | No | Pagination |

**Response** `200 OK`

```json
{
  "count": 6,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": 1,
      "pipeline": 1,
      "code": "new",
      "name": "New",
      "probability": 10,
      "sort_order": 0,
      "is_won": false,
      "is_lost": false,
      "metadata": {},
      "created_at": "2026-01-15T00:00:00Z",
      "updated_at": "2026-01-15T00:00:00Z"
    }
  ]
}
```

```bash
curl -X GET "https://platform.iblai.app/api/crm/pipelines/1/stages/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/pipelines/{pipeline_id}/stages/`

Create a stage in this pipeline.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | Yes | Stable identifier referenced by `Deal.stage` payloads. Unique within the pipeline |
| `name` | string | Yes | Display name shown on Kanban boards |
| `probability` | integer | No | Win likelihood at this stage, 0–100 (defaults to `0`). Used for revenue forecasting |
| `sort_order` | integer | No | Left-to-right position on the Kanban board (defaults to `0`) |
| `is_won` | boolean | No | Terminal won stage. Moving a Deal here sets `Deal.status='won'`. Defaults to `false` |
| `is_lost` | boolean | No | Terminal lost stage. Moving a Deal here sets `Deal.status='lost'`. Defaults to `false` |
| `metadata` | object | No | Free-form JSON (defaults to `{}`) |

A stage cannot be both `is_won` and `is_lost`.

```json
{
  "code": "discovery",
  "name": "Discovery",
  "probability": 15,
  "sort_order": 1,
  "is_won": false,
  "is_lost": false
}
```

**Response** `201 Created` — full stage object (`pipeline` is set automatically from the URL).

**Error** `400 Bad Request` — duplicate code.

```json
{
  "code": [
    "A stage with code 'discovery' already exists in this Pipeline."
  ]
}
```

**Error** `400 Bad Request` — probability out of range.

```json
{"probability": ["probability must be between 0 and 100."]}
```

**Error** `400 Bad Request` — both terminal flags set.

```json
{"detail": ["A stage cannot be both is_won and is_lost."]}
```

**Error** `404 Not Found` — parent pipeline does not exist or belongs to another Platform.

```bash
curl -X POST "https://platform.iblai.app/api/crm/pipelines/1/stages/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "discovery",
    "name": "Discovery",
    "probability": 15,
    "sort_order": 1
  }'
```

---

#### GET `/pipelines/{pipeline_id}/stages/{id}/`

Retrieve one stage. 404 if the stage is not part of the URL pipeline.

**Response** `200 OK` — full stage object.

```bash
curl -X GET "https://platform.iblai.app/api/crm/pipelines/1/stages/2/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### PUT `/pipelines/{pipeline_id}/stages/{id}/` and PATCH `/pipelines/{pipeline_id}/stages/{id}/`

Replace or update a stage. `pipeline` is read-only — stages cannot be moved between pipelines.

```json
{
  "name": "Discovery Call",
  "probability": 20
}
```

**Response** `200 OK` — full updated stage object.

```bash
curl -X PATCH "https://platform.iblai.app/api/crm/pipelines/1/stages/2/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Discovery Call", "probability": 20}'
```

---

#### DELETE `/pipelines/{pipeline_id}/stages/{id}/`

Delete a stage.

**Response** `204 No Content`.

**Error** `409 Conflict` — at least one Deal still references this stage. Move those deals to a different stage first.

```json
{"detail": "Stage still has Deals attached."}
```

**Error** `404 Not Found`.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/pipelines/1/stages/2/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

### 7.4 Lead Sources

Lead Sources record where a Deal originated (web traffic, a referral, an outbound campaign, …). They live under the `Ibl.CRM/Pipelines/` RBAC bucket — anyone who can shape pipelines can shape lead sources.

**Seeded defaults.** Every Platform ships with four lead sources, created automatically on Platform creation and backfilled into existing Platforms by a data migration:

| `code` | `name` |
|--------|--------|
| `web` | Web |
| `referral` | Referral |
| `cold_call` | Cold Call |
| `advertisement` | Advertisement |

#### Summary

| Method | Endpoint | Auth | RBAC Permission | Description |
|--------|----------|------|-----------------|-------------|
| GET    | `/lead-sources/`        | Token | `Ibl.CRM/Pipelines/list`   | Paginated list of lead sources |
| POST   | `/lead-sources/`        | Token | `Ibl.CRM/Pipelines/action` | Create a lead source |
| GET    | `/lead-sources/{id}/`   | Token | `Ibl.CRM/Pipelines/read`   | Retrieve one lead source |
| PUT    | `/lead-sources/{id}/`   | Token | `Ibl.CRM/Pipelines/write`  | Replace all editable fields |
| PATCH  | `/lead-sources/{id}/`   | Token | `Ibl.CRM/Pipelines/write`  | Update a subset of fields |
| DELETE | `/lead-sources/{id}/`   | Token | `Ibl.CRM/Pipelines/delete` | Delete a lead source (Deals referencing it have `source` cleared) |

---

#### GET `/lead-sources/`

List lead sources in your Platform. Paginated.

**Query parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `code` | string | No | Case-insensitive exact match |
| `name` | string | No | Case-insensitive substring match |
| `page` | integer | No | Page number |
| `page_size` | integer | No | Rows per page (max 100) |

**Response** `200 OK`

```json
{
  "count": 5,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": 1,
      "platform": 1,
      "name": "Web",
      "code": "web",
      "metadata": {},
      "created_at": "2026-01-15T00:00:00Z",
      "updated_at": "2026-01-15T00:00:00Z"
    },
    {
      "id": 2,
      "platform": 1,
      "name": "Referral",
      "code": "referral",
      "metadata": {},
      "created_at": "2026-01-15T00:00:00Z",
      "updated_at": "2026-01-15T00:00:00Z"
    },
    {
      "id": 3,
      "platform": 1,
      "name": "Cold Call",
      "code": "cold_call",
      "metadata": {},
      "created_at": "2026-01-15T00:00:00Z",
      "updated_at": "2026-01-15T00:00:00Z"
    },
    {
      "id": 4,
      "platform": 1,
      "name": "Advertisement",
      "code": "advertisement",
      "metadata": {},
      "created_at": "2026-01-15T00:00:00Z",
      "updated_at": "2026-01-15T00:00:00Z"
    },
    {
      "id": 5,
      "platform": 1,
      "name": "LinkedIn Outbound",
      "code": "linkedin-outbound",
      "metadata": {"campaign": "q3-saas"},
      "created_at": "2026-06-01T10:00:00Z",
      "updated_at": "2026-06-01T10:00:00Z"
    }
  ]
}
```

**Response fields**

| Field | Type | Description |
|-------|------|-------------|
| `results[].id` | integer | Server-assigned. Stable across renames |
| `results[].platform` | string | Platform key. Set server-side from the auth token |
| `results[].name` | string | Display name shown to sales reps when categorizing a Deal |
| `results[].code` | string | Stable slug used in API payloads and reports. Unique per Platform; lowercase letters, digits, hyphens |
| `results[].metadata` | object | Free-form JSON |
| `results[].created_at` / `updated_at` | ISO 8601 | Timestamps |

```bash
curl -X GET "https://platform.iblai.app/api/crm/lead-sources/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/lead-sources/`

Create a lead source.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Display name |
| `code` | string | Yes | Stable slug. Unique per Platform; lowercase letters, digits, hyphens |
| `metadata` | object | No | Free-form JSON (defaults to `{}`) |

```json
{
  "name": "LinkedIn Outbound",
  "code": "linkedin-outbound",
  "metadata": {"campaign": "q3-saas"}
}
```

**Response** `201 Created` — full lead source object.

**Error** `400 Bad Request` — duplicate code.

```json
{
  "code": [
    "A LeadSource with code 'linkedin-outbound' already exists on this Platform."
  ]
}
```

**Error** `403 Forbidden` — caller missing `Ibl.CRM/Pipelines/action`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/lead-sources/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "LinkedIn Outbound",
    "code": "linkedin-outbound",
    "metadata": {"campaign": "q3-saas"}
  }'
```

---

#### GET `/lead-sources/{id}/`

Retrieve one lead source.

**Response** `200 OK` — full lead source object.

**Error** `404 Not Found`.

```bash
curl -X GET "https://platform.iblai.app/api/crm/lead-sources/5/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### PUT `/lead-sources/{id}/` and PATCH `/lead-sources/{id}/`

Replace or update a lead source.

```json
{"name": "LinkedIn — Outbound (NA)"}
```

**Response** `200 OK` — full updated lead source object.

```bash
curl -X PATCH "https://platform.iblai.app/api/crm/lead-sources/5/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "LinkedIn — Outbound (NA)"}'
```

---

#### DELETE `/lead-sources/{id}/`

Delete a lead source. **This is not blocked by attached Deals** — unlike Pipelines and Stages, which return `409 Conflict` when Deals reference them, deleting a Lead Source clears the `source` foreign key on every Deal that referenced it (`SET NULL`). Treat this as destructive: once the source is gone, the historical "where did this Deal come from?" attribution is lost.

**Response** `204 No Content`.

**Error** `404 Not Found`.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/lead-sources/5/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---
### 7.5 Deals

A Deal is a sales opportunity moving through a Pipeline. Deals carry monetary value, a current Stage, an optional Person and Organization, an owner, a Lead Source, and a free-form metadata bag. Lifecycle (`status`, `closed_at`) is service-managed: clients never set it directly — they call `move-stage/`, `won/`, or `lost/` and the service derives status from the destination Stage's `is_won` / `is_lost` flags.

Tags are attached and detached via dedicated sub-routes that live on the Deal viewset but are governed by `Ibl.CRM/Tags/write` — see 7.7.

#### Endpoint summary

| Method | Path | Purpose | Required permission |
|---|---|---|---|
| GET | `/deals/` | List deals (paginated) | `Ibl.CRM/Deals/list` |
| POST | `/deals/` | Create a deal | `Ibl.CRM/Deals/action` |
| GET | `/deals/{id}/` | Retrieve one deal | `Ibl.CRM/Deals/read` |
| PUT | `/deals/{id}/` | Replace all editable fields | `Ibl.CRM/Deals/write` |
| PATCH | `/deals/{id}/` | Patch supplied fields | `Ibl.CRM/Deals/write` |
| DELETE | `/deals/{id}/` | Delete the deal | `Ibl.CRM/Deals/delete` |
| POST | `/deals/{id}/move-stage/` | Move to a specific stage | `Ibl.CRM/Deals/write` |
| POST | `/deals/{id}/won/` | Mark won (move to first `is_won` stage) | `Ibl.CRM/Deals/write` |
| POST | `/deals/{id}/lost/` | Mark lost (requires `lost_reason`) | `Ibl.CRM/Deals/write` |
| POST | `/deals/{id}/tags/` | Attach a tag | `Ibl.CRM/Tags/write` |
| DELETE | `/deals/{id}/tags/{tag_id}/` | Detach a tag | `Ibl.CRM/Tags/write` |

All responses are scoped to the Platform on your access token. A deal that belongs to a different Platform returns `404 Not Found` — never `403` — so the API never leaks the existence of records outside your scope.

---

#### GET `/deals/`

List deals in your Platform. Paginated.

**Query parameters**

| Param | Type | Description |
|---|---|---|
| `status` | string | One of `open`, `won`, `lost` |
| `pipeline` | integer | Pipeline id |
| `stage` | integer | PipelineStage id |
| `owner` | integer | Owning user id |
| `source` | integer | LeadSource id |
| `person` | UUID | Primary person on the deal |
| `organization` | UUID | Organization on the deal |
| `expected_close_date__gte` | ISO-8601 datetime | Forecast close on or after |
| `expected_close_date__lte` | ISO-8601 datetime | Forecast close on or before |
| `created_at__gte` | ISO-8601 datetime | Created on or after |
| `created_at__lte` | ISO-8601 datetime | Created on or before |
| `metadata__has_key` | string | Only deals whose `metadata` object contains this top-level key |
| `tags` | integer (repeatable) or CSV | OR-match on tag ids. Accepts `?tags=1&tags=2` or `?tags=1,2`. Non-numeric pieces are dropped silently |
| `page` | integer | Page number, defaults to 1 |
| `page_size` | integer | Items per page |

**Response** `200 OK`

```json
{
  "count": 47,
  "next_page": 2,
  "previous_page": null,
  "results": [
    {
      "id": 184,
      "platform": 1,
      "title": "Acme renewal — 2026",
      "description": "Multi-year renewal, 220 seats.",
      "lead_value": "48000.00",
      "currency": "USD",
      "status": "open",
      "lost_reason": "",
      "expected_close_date": "2026-09-30T00:00:00Z",
      "closed_at": null,
      "person": "c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80",
      "organization": "b2a1e7d3-6f4c-4b2a-8e9d-1c3a5b7d9f10",
      "pipeline": 3,
      "stage": 12,
      "source": 5,
      "owner": 91,
      "tags": [
        {"id": 7, "name": "Enterprise", "color": "#3F6BFF"},
        {"id": 11, "name": "Renewal", "color": "#22A06B"}
      ],
      "metadata": {"region": "NA", "campaign_id": "fy26-renewals"},
      "created_at": "2026-04-12T08:14:22Z",
      "updated_at": "2026-05-30T17:02:11Z"
    }
  ]
}
```

**Response fields**

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Server-assigned |
| `platform` | integer | Owning Platform id (always your Platform on read) |
| `title` | string | Short display name |
| `description` | string | Free-text notes |
| `lead_value` | decimal string | Estimated value in `currency` |
| `currency` | string | ISO 4217 currency code, defaults to `"USD"` |
| `status` | string | **Read-only.** `open`, `won`, or `lost`. Derived from the current Stage |
| `lost_reason` | string | Persisted by the `lost/` action; empty otherwise |
| `expected_close_date` | datetime\|null | Forecasted close date |
| `closed_at` | datetime\|null | **Read-only.** Stamped on entry to a `won`/`lost` stage; cleared on re-open |
| `person` | UUID | Primary person on the deal |
| `organization` | UUID\|null | Organization the deal is with |
| `pipeline` | integer | Pipeline this deal flows through |
| `stage` | integer | Current PipelineStage |
| `source` | integer\|null | LeadSource id |
| `owner` | integer\|null | Owning user id |
| `tags` | array | Flat list of `{id, name, color}` chips (see 7.7) |
| `metadata` | object | Free-form JSON |
| `created_at` | datetime | Creation timestamp |
| `updated_at` | datetime | Last-modified timestamp |

```bash
curl -X GET "https://platform.iblai.app/api/crm/deals/?status=open&pipeline=3&tags=7&tags=11&page=1" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/deals/`

Create a deal.

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | Yes | Short display name |
| `pipeline` | integer | Yes | Pipeline id; must belong to your Platform |
| `stage` | integer | Yes | Initial PipelineStage; must belong to `pipeline` |
| `person` | UUID | Yes | Primary person on the deal; must belong to your Platform |
| `description` | string | No | Free-text notes, defaults to `""` |
| `lead_value` | decimal string | No | Estimated value, defaults to `"0"` |
| `currency` | string | No | ISO 4217 code, defaults to `"USD"` |
| `expected_close_date` | datetime | No | Forecasted close date |
| `organization` | UUID | No | Must belong to your Platform. If the person already has an organization, this must match it |
| `source` | integer | No | LeadSource id; must belong to your Platform |
| `owner` | integer | No | Owning user. Defaults to the calling user. Must be an active member of your Platform |
| `lost_reason` | string | No | Pre-set lost reason; usually populated by the `lost/` action instead |
| `metadata` | object | No | Free-form JSON, defaults to `{}` |

Do **not** send `status` or `closed_at` — these are service-managed (see footgun below).

```json
{
  "title": "Initech expansion — Q3",
  "description": "Add 40 seats + analytics module.",
  "lead_value": "18500.00",
  "currency": "USD",
  "expected_close_date": "2026-08-31T00:00:00Z",
  "person": "c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80",
  "organization": "b2a1e7d3-6f4c-4b2a-8e9d-1c3a5b7d9f10",
  "pipeline": 3,
  "stage": 11,
  "source": 5,
  "owner": 91,
  "metadata": {"region": "NA"}
}
```

**Response** `201 Created` — full deal object (same shape as the list result row).

**Error** `400 Bad Request` — validation failed.

Examples of validation errors you may see:

```json
{"status": "Service-managed — write via `POST /deals/{id}/move-stage/`, `won/`, or `lost/`."}
```

```json
{"stage": "Stage does not belong to the supplied pipeline."}
```

```json
{"organization": "Organization does not match the Person's Organization."}
```

```json
{"person": "Person belongs to a different platform."}
```

```json
{"owner": "User is not an active member of this Platform."}
```

**Error** `403 Forbidden` — caller missing `Ibl.CRM/Deals/action`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/deals/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Initech expansion — Q3",
    "lead_value": "18500.00",
    "currency": "USD",
    "person": "c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80",
    "organization": "b2a1e7d3-6f4c-4b2a-8e9d-1c3a5b7d9f10",
    "pipeline": 3,
    "stage": 11,
    "source": 5,
    "metadata": {"region": "NA"}
  }'
```

---

#### GET `/deals/{id}/`

Retrieve one deal.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | integer | Deal id |

**Response** `200 OK` — full deal object.

**Error** `404 Not Found`.

```bash
curl -X GET "https://platform.iblai.app/api/crm/deals/184/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### PUT `/deals/{id}/` and PATCH `/deals/{id}/`

Replace (PUT) or patch (PATCH) editable fields on a deal. `status` and `closed_at` are read-only on the serializer; sending either returns `400`.

```json
{
  "title": "Acme renewal — 2026 (NA + EMEA)",
  "lead_value": "62000.00",
  "expected_close_date": "2026-10-15T00:00:00Z",
  "metadata": {"region": "NA+EMEA", "campaign_id": "fy26-renewals"}
}
```

**Response** `200 OK` — full updated deal object.

**Error** `400 Bad Request` — validation failed. The most common cause is trying to write `status` or `closed_at`:

```json
{
  "status": "Service-managed — write via `POST /deals/{id}/move-stage/`, `won/`, or `lost/`.",
  "closed_at": "Service-managed — write via `POST /deals/{id}/move-stage/`, `won/`, or `lost/`."
}
```

**Error** `403 Forbidden` — missing `Ibl.CRM/Deals/write`.

```bash
curl -X PATCH "https://platform.iblai.app/api/crm/deals/184/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"lead_value": "62000.00"}'
```

---

#### DELETE `/deals/{id}/`

Delete a deal. Cascades to the deal's audit Activities and tag assignments.

**Response** `204 No Content`.

**Error** `404 Not Found`.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/deals/184/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/deals/{id}/move-stage/`

Move a deal to a different stage in its current pipeline.

**Request body** — provide **either** `stage_id` **or** `stage_code`:

| Field | Type | Required | Description |
|---|---|---|---|
| `stage_id` | integer | One of `stage_id` / `stage_code` | Destination PipelineStage id |
| `stage_code` | string | One of `stage_id` / `stage_code` | Stable `code` of the destination stage within the deal's pipeline |

`stage_code` is the integration-safe choice across environments where ids differ but codes are stable.

```json
{"stage_code": "negotiation"}
```

**Response** `200 OK` — full deal object with updated `stage` and recomputed `status` / `closed_at`.

**Side effects on a successful, non-no-op move:**

1. `Deal.status` is recomputed from the destination stage (`is_won` → `won`, `is_lost` → `lost`, otherwise `open`).
2. `Deal.closed_at` is stamped on entry to a terminal stage and cleared when re-opening.
3. An audit Activity is appended:
   - `type`: `note`
   - `title`: `"Stage changed"`
   - `comment`: `"<from-stage-name> → <to-stage-name>"`
   - `is_done`: `true` with `done_at` set to now
   - `owner`: the calling user (or null for service-account callers without an underlying user)
   - `deal`, `person`: copied from the deal
4. The `deal_stage_changed` signal fires, dispatching a `CRM_DEAL_STAGE_CHANGED` notification (Section 12).

**Footgun — moving to the current stage is a no-op.** The service detects `from_stage == to_stage` and returns the deal unchanged: no save, no signal, no audit Activity. If your UI relies on a returned `updated_at` change to confirm the move, gate the call on a real stage delta first.

**Footgun — concurrent moves serialise on the row.** The deal is re-read under a `SELECT FOR UPDATE` so two simultaneous transitions (e.g. a kanban drag landing at the same instant as a `won/` button click) cannot both observe the same `from_stage`. The second transaction sees the post-state of the first.

**Error** `400 Bad Request`

```json
{"detail": "Provide either `stage_id` or `stage_code`."}
```

```json
{"detail": "Destination stage does not belong to this Deal's pipeline."}
```

**Error** `404 Not Found` — deal or stage not found, or stage lives on a pipeline owned by another Platform.

```json
{"detail": "Stage 'negotiation' not found in Deal's pipeline."}
```

**Error** `403 Forbidden` — missing `Ibl.CRM/Deals/write`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/deals/184/move-stage/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"stage_code": "negotiation"}'
```

---

#### POST `/deals/{id}/won/`

Mark a deal as won. Moves it into a closed-won stage and sets `status="won"`.

**Request body** — all fields optional:

| Field | Type | Required | Description |
|---|---|---|---|
| `stage_code` | string | No | Specific `is_won=true` stage code in the deal's pipeline. When omitted, the first `is_won=true` stage by `sort_order` is used |

```json
{}
```

```json
{"stage_code": "closed-won-expansion"}
```

**Response** `200 OK` — full deal object. The same side effects as `move-stage/` apply (audit Activity, `deal_stage_changed`), **plus** a `deal_won` signal.

**Error** `400 Bad Request`

```json
{"detail": "Pipeline 3 has no is_won=True stage configured."}
```

```json
{"detail": "Stage 'closed-won-expansion' is not marked is_won=True."}
```

**Error** `403 Forbidden` — missing `Ibl.CRM/Deals/write`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/deals/184/won/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

#### POST `/deals/{id}/lost/`

Mark a deal as lost. Moves it into a closed-lost stage, persists `lost_reason` on the deal, and sets `status="lost"`.

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `lost_reason` | string | **Yes** | Free-text reason persisted on `Deal.lost_reason`. Max 255 characters, must not be blank |
| `stage_code` | string | No | Specific `is_lost=true` stage code in the deal's pipeline. When omitted, the first `is_lost=true` stage by `sort_order` is used |

```json
{
  "lost_reason": "Chose competitor — Pied Piper undercut us on price.",
  "stage_code": "closed-lost-price"
}
```

**Response** `200 OK` — full deal object. The same side effects as `move-stage/` apply, **plus** a `deal_lost` signal with the `lost_reason` payload.

**Error** `400 Bad Request`

```json
{"lost_reason": ["This field is required."]}
```

```json
{"detail": "Pipeline 3 has no is_lost=True stage configured."}
```

```json
{"detail": "Stage 'closed-lost-price' is not marked is_lost=True."}
```

**Error** `403 Forbidden` — missing `Ibl.CRM/Deals/write`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/deals/184/lost/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"lost_reason": "Chose competitor — Pied Piper undercut us on price."}'
```

---

#### POST `/deals/{id}/tags/` and DELETE `/deals/{id}/tags/{tag_id}/`

Attach or detach a tag on a deal. The endpoints are documented in 7.7 alongside the Person and Organization variants — the contract is identical across host types. The required permission is `Ibl.CRM/Tags/write`; holding `Ibl.CRM/Deals/write` is **not** sufficient.

---

### 7.6 Activities

An Activity is anything that goes on a deal or person's timeline: a logged call, a scheduled meeting, an email, a free-text note, a task, a deadline, or a lunch. Each activity attaches to a Deal **or** a Person (or both, if they're consistent) and carries optional schedule fields, a reminder, an owner, and a free-form metadata bag.

Activities are also the audit row that stage transitions on Deals leave behind — the `move-stage/`, `won/`, and `lost/` actions all write an `Activity(type="note", title="Stage changed", is_done=true)` automatically. Those rows are indistinguishable from user-authored notes other than their title and comment shape.

#### Endpoint summary

| Method | Path | Purpose | Required permission |
|---|---|---|---|
| GET | `/activities/` | List activities (paginated) | `Ibl.CRM/Activities/list` |
| POST | `/activities/` | Create an activity | `Ibl.CRM/Activities/action` |
| GET | `/activities/{id}/` | Retrieve one activity | `Ibl.CRM/Activities/read` |
| PUT | `/activities/{id}/` | Replace all editable fields | `Ibl.CRM/Activities/write` |
| PATCH | `/activities/{id}/` | Patch supplied fields | `Ibl.CRM/Activities/write` |
| DELETE | `/activities/{id}/` | Delete the activity | `Ibl.CRM/Activities/delete` |
| POST | `/activities/{id}/done/` | Mark done (idempotent) | `Ibl.CRM/Activities/write` |

---

#### GET `/activities/`

List activities in your Platform. Paginated.

**Query parameters**

| Param | Type | Description |
|---|---|---|
| `type` | string | One of `call`, `meeting`, `email`, `note`, `task`, `lunch`, `deadline` |
| `is_done` | boolean | Filter completed vs open |
| `owner` | integer | Owning user id |
| `deal` | integer | Deal id |
| `person` | UUID | Person id |
| `schedule_from__gte` | ISO-8601 datetime | Scheduled start on or after |
| `schedule_from__lte` | ISO-8601 datetime | Scheduled start on or before |
| `metadata__has_key` | string | Only activities whose `metadata` object contains this top-level key |
| `page` | integer | Page number, defaults to 1 |
| `page_size` | integer | Items per page |

To render a deal's timeline, filter by `deal` and order by `created_at` descending (default ordering — see the resource reference in section 6). To render a person's timeline, filter by `person`.

**Response** `200 OK`

```json
{
  "count": 12,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": 4421,
      "platform": 1,
      "title": "Stage changed",
      "type": "note",
      "location": "",
      "comment": "Discovery → Proposal",
      "schedule_from": null,
      "schedule_to": null,
      "is_done": true,
      "done_at": "2026-05-30T17:02:11Z",
      "deal": 184,
      "person": "c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80",
      "owner": 91,
      "reminder_at": null,
      "reminder_sent": false,
      "metadata": {},
      "created_at": "2026-05-30T17:02:11Z",
      "updated_at": "2026-05-30T17:02:11Z"
    },
    {
      "id": 4408,
      "platform": 1,
      "title": "Discovery call",
      "type": "call",
      "location": "Zoom",
      "comment": "Walk through requirements with VP Eng.",
      "schedule_from": "2026-06-02T15:00:00Z",
      "schedule_to": "2026-06-02T15:45:00Z",
      "is_done": false,
      "done_at": null,
      "deal": 184,
      "person": "c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80",
      "owner": 91,
      "reminder_at": "2026-06-02T14:45:00Z",
      "reminder_sent": false,
      "metadata": {"meeting_link": "https://zoom.us/j/123"},
      "created_at": "2026-05-29T11:18:02Z",
      "updated_at": "2026-05-29T11:18:02Z"
    }
  ]
}
```

**Response fields**

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Server-assigned |
| `platform` | integer | Owning Platform id |
| `title` | string | Short summary |
| `type` | string | One of `call`, `meeting`, `email`, `note`, `task`, `lunch`, `deadline` |
| `location` | string | Meeting location, dial-in, venue |
| `comment` | string | Free-text notes |
| `schedule_from` | datetime\|null | Scheduled start |
| `schedule_to` | datetime\|null | Scheduled end |
| `is_done` | boolean | Completion flag |
| `done_at` | datetime\|null | **Read-only.** Stamped the first time `is_done` flips true |
| `deal` | integer\|null | Attached deal id |
| `person` | UUID\|null | Attached person id |
| `owner` | integer\|null | Owning user id |
| `reminder_at` | datetime\|null | When to remind the owner |
| `reminder_sent` | boolean | **Read-only.** Flipped true after the reminder dispatch runs |
| `metadata` | object | Free-form JSON |
| `created_at` | datetime | Creation timestamp |
| `updated_at` | datetime | Last-modified timestamp |

```bash
curl -X GET "https://platform.iblai.app/api/crm/activities/?deal=184&is_done=false" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/activities/`

Create an activity.

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | Yes | Short summary |
| `type` | string | Yes | One of `call`, `meeting`, `email`, `note`, `task`, `lunch`, `deadline` |
| `deal` | integer | Yes (or `person`) | Deal id; must belong to your Platform |
| `person` | UUID | Yes (or `deal`) | Person id; must belong to your Platform |
| `location` | string | No | Defaults to `""` |
| `comment` | string | No | Defaults to `""` |
| `schedule_from` | datetime | No | Scheduled start |
| `schedule_to` | datetime | No | Scheduled end |
| `is_done` | boolean | No | Defaults to `false`. Prefer `POST /activities/{id}/done/` to flip this |
| `owner` | integer | No | Defaults to the calling user. Must be an active member of your Platform |
| `reminder_at` | datetime | No | When to remind the owner |
| `metadata` | object | No | Free-form JSON, defaults to `{}` |

**Attachment rule.** An activity must attach to a `deal` **or** a `person` (or both). When both are supplied, the person must equal the deal's `person` — attaching an activity to Deal A while tagging Person B is rejected because it splits the audit trail across unrelated records.

**Schedule semantics.** The two schedule fields encode three timeline shapes:

| `schedule_from` | `schedule_to` | Read as |
|---|---|---|
| null | null | A past log entry — something that already happened (e.g. "Logged call") |
| set | null | A scheduled task or deadline with a start time but no fixed end |
| set | set | A meeting / time-bounded event |

The API does not enforce ordering between `schedule_from` and `schedule_to` — that validation is a UI concern.

```json
{
  "title": "Discovery call",
  "type": "call",
  "location": "Zoom",
  "comment": "Walk through requirements with VP Eng.",
  "schedule_from": "2026-06-02T15:00:00Z",
  "schedule_to": "2026-06-02T15:45:00Z",
  "deal": 184,
  "person": "c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80",
  "reminder_at": "2026-06-02T14:45:00Z",
  "metadata": {"meeting_link": "https://zoom.us/j/123"}
}
```

**Response** `201 Created` — full activity object.

**Error** `400 Bad Request`

```json
{"detail": "Activity must attach to a `deal` or a `person`."}
```

```json
{"person": "Person does not match the Deal's Person."}
```

```json
{"deal": "Deal belongs to a different platform."}
```

```json
{"owner": "User is not an active member of this Platform."}
```

**Error** `403 Forbidden` — caller missing `Ibl.CRM/Activities/action`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/activities/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Discovery call",
    "type": "call",
    "location": "Zoom",
    "schedule_from": "2026-06-02T15:00:00Z",
    "schedule_to": "2026-06-02T15:45:00Z",
    "deal": 184,
    "person": "c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80",
    "reminder_at": "2026-06-02T14:45:00Z"
  }'
```

---

#### GET `/activities/{id}/`

Retrieve one activity.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | integer | Activity id |

**Response** `200 OK` — full activity object.

**Error** `404 Not Found`.

```bash
curl -X GET "https://platform.iblai.app/api/crm/activities/4408/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### PUT `/activities/{id}/` and PATCH `/activities/{id}/`

Replace or patch editable fields on an activity. `done_at` and `reminder_sent` are read-only.

The deal/person attachment rule (must have one; if both, they must agree) is re-checked on every write: the values are merged from the supplied payload onto the stored row before validation runs, so a PATCH that clears `deal` on a person-less activity returns `400`.

```json
{
  "title": "Discovery call — rescheduled",
  "schedule_from": "2026-06-03T15:00:00Z",
  "schedule_to": "2026-06-03T15:45:00Z"
}
```

**Response** `200 OK` — full updated activity object.

**Error** `400 Bad Request` — see attachment validation errors above.

**Error** `403 Forbidden` — missing `Ibl.CRM/Activities/write`.

```bash
curl -X PATCH "https://platform.iblai.app/api/crm/activities/4408/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"schedule_from": "2026-06-03T15:00:00Z", "schedule_to": "2026-06-03T15:45:00Z"}'
```

---

#### DELETE `/activities/{id}/`

Delete an activity.

**Be deliberate when deleting auto-generated stage-change rows** (`title="Stage changed"`, `type="note"`). They are the audit trail for `move-stage/`, `won/`, and `lost/` transitions; once deleted there is no way to reconstruct the transition history of a deal.

**Response** `204 No Content`.

**Error** `404 Not Found`.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/activities/4408/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/activities/{id}/done/`

Mark an activity as done. Flips `is_done=true` and stamps `done_at` if it isn't already set.

**Request body** — none. Any payload is ignored.

**Idempotent.** Calling `done/` on an already-done activity is a safe no-op: the original `done_at` is preserved. This makes the action safe to retry from a flaky network without worrying about overwriting completion timestamps that downstream reports depend on.

**Response** `200 OK` — full activity object with `is_done=true` and `done_at` populated.

```json
{
  "id": 4408,
  "platform": 1,
  "title": "Discovery call",
  "type": "call",
  "location": "Zoom",
  "comment": "Walk through requirements with VP Eng.",
  "schedule_from": "2026-06-02T15:00:00Z",
  "schedule_to": "2026-06-02T15:45:00Z",
  "is_done": true,
  "done_at": "2026-06-02T15:48:13Z",
  "deal": 184,
  "person": "c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80",
  "owner": 91,
  "reminder_at": "2026-06-02T14:45:00Z",
  "reminder_sent": true,
  "metadata": {"meeting_link": "https://zoom.us/j/123"},
  "created_at": "2026-05-29T11:18:02Z",
  "updated_at": "2026-06-02T15:48:13Z"
}
```

**Error** `403 Forbidden` — missing `Ibl.CRM/Activities/write`.

**Error** `404 Not Found`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/activities/4408/done/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

### 7.7 Tags

A Tag is a colored label scoped to your Platform. Tags are attached to host records — Persons, Organizations, and Deals — through dedicated assignment tables, so each host type has its own attach / detach pair of endpoints (documented in 7.1, 7.2, and 7.5). This section covers the Tag rows themselves: CRUD plus the universal attach/detach contract.

The mental model: **Tag rows are the taxonomy; assignments are the chips you see on a record.** A single Tag named `Enterprise` lives once and can be attached to many records of different host types.

#### Endpoint summary

| Method | Path | Purpose | Required permission |
|---|---|---|---|
| GET | `/tags/` | List tags (paginated) | `Ibl.CRM/Tags/list` |
| POST | `/tags/` | Create a tag | `Ibl.CRM/Tags/action` |
| GET | `/tags/{id}/` | Retrieve one tag | `Ibl.CRM/Tags/read` |
| PUT | `/tags/{id}/` | Replace all editable fields | `Ibl.CRM/Tags/write` |
| PATCH | `/tags/{id}/` | Patch supplied fields | `Ibl.CRM/Tags/write` |
| DELETE | `/tags/{id}/` | Delete the tag — cascades to every host | `Ibl.CRM/Tags/delete` |

Attach / detach lives on the host viewset:

| Method | Path | Purpose | Required permission |
|---|---|---|---|
| POST | `/persons/{person_id}/tags/` | Attach tag to a person | `Ibl.CRM/Tags/write` |
| DELETE | `/persons/{person_id}/tags/{tag_id}/` | Detach tag from a person | `Ibl.CRM/Tags/write` |
| POST | `/organizations/{organization_id}/tags/` | Attach tag to an organization | `Ibl.CRM/Tags/write` |
| DELETE | `/organizations/{organization_id}/tags/{tag_id}/` | Detach tag from an organization | `Ibl.CRM/Tags/write` |
| POST | `/deals/{deal_id}/tags/` | Attach tag to a deal | `Ibl.CRM/Tags/write` |
| DELETE | `/deals/{deal_id}/tags/{tag_id}/` | Detach tag from a deal | `Ibl.CRM/Tags/write` |

**Critical permission rule.** All six attach/detach routes are gated by `Ibl.CRM/Tags/write`. Holding the host-side write action — `Ibl.CRM/Persons/write`, `Ibl.CRM/Organizations/write`, or `Ibl.CRM/Deals/write` — does **not** allow you to tag the record. A sales rep with full deal-edit rights but no tag-write rights will still get `403 Forbidden` from `POST /deals/{id}/tags/`. This is intentional: tag taxonomy is a Platform-admin concern, not a per-record editing concern.

---

#### GET `/tags/`

List tags in your Platform. Paginated.

**Query parameters**

| Param | Type | Description |
|---|---|---|
| `name` | string | Case-insensitive substring match |
| `created_at__gte` | ISO-8601 datetime | Created on or after |
| `created_at__lte` | ISO-8601 datetime | Created on or before |
| `page` | integer | Page number, defaults to 1 |
| `page_size` | integer | Items per page |

**Response** `200 OK`

```json
{
  "count": 14,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": 7,
      "platform": 1,
      "name": "Enterprise",
      "color": "#3F6BFF",
      "metadata": {"order": 1},
      "created_at": "2026-02-08T09:12:01Z",
      "updated_at": "2026-02-08T09:12:01Z"
    },
    {
      "id": 11,
      "platform": 1,
      "name": "Renewal",
      "color": "#22A06B",
      "metadata": {},
      "created_at": "2026-02-08T09:14:33Z",
      "updated_at": "2026-04-19T18:01:44Z"
    }
  ]
}
```

**Response fields**

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Server-assigned |
| `platform` | integer | Owning Platform id |
| `name` | string | Display label, up to 64 characters, unique per Platform (case-sensitive) |
| `color` | string | Hex string of the form `#RRGGBB`. Six hex digits, leading `#`. Defaults to `#888888` |
| `metadata` | object | Free-form JSON |
| `created_at` | datetime | Creation timestamp |
| `updated_at` | datetime | Last-modified timestamp |

```bash
curl -X GET "https://platform.iblai.app/api/crm/tags/?name=enterprise" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/tags/`

Create a tag.

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Up to 64 characters. Trimmed of leading/trailing whitespace. Must be unique within your Platform (case-sensitive after trim) |
| `color` | string | No | Hex `#RRGGBB`. Defaults to `#888888`. Six hex digits required — `#FFF` shorthand is rejected |
| `metadata` | object | No | Free-form JSON, defaults to `{}` |

```json
{
  "name": "Enterprise",
  "color": "#3F6BFF",
  "metadata": {"order": 1}
}
```

**Response** `201 Created` — full tag object.

**Error** `400 Bad Request`

```json
{"name": ["A Tag with this name already exists in this Platform."]}
```

```json
{"name": ["Name must not be blank."]}
```

```json
{"name": ["Name must be at most 64 characters."]}
```

```json
{"color": ["Color must be a hex string like `#3F6BFF`."]}
```

**Error** `403 Forbidden` — caller missing `Ibl.CRM/Tags/action`.

```bash
curl -X POST "https://platform.iblai.app/api/crm/tags/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Enterprise", "color": "#3F6BFF"}'
```

---

#### GET `/tags/{id}/`

Retrieve one tag.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | integer | Tag id |

**Response** `200 OK` — full tag object.

**Error** `404 Not Found`.

```bash
curl -X GET "https://platform.iblai.app/api/crm/tags/7/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### PUT `/tags/{id}/` and PATCH `/tags/{id}/`

Replace or patch a tag — typically `name` or `color`.

Renames take effect immediately on every host that carries the tag chip — chips render `{id, name, color}` live from the Tag row, so a rename updates everywhere the tag appears without touching the assignment tables.

```json
{"name": "Enterprise (Tier 1)", "color": "#1F4FE6"}
```

**Response** `200 OK` — full updated tag object.

**Error** `400 Bad Request` — duplicate name within your Platform, blank name, name longer than 64 characters, or bad color hex (see POST examples above).

**Error** `403 Forbidden` — missing `Ibl.CRM/Tags/write`.

```bash
curl -X PATCH "https://platform.iblai.app/api/crm/tags/7/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Enterprise (Tier 1)", "color": "#1F4FE6"}'
```

---

#### DELETE `/tags/{id}/`

Delete a tag.

> **DESTRUCTIVE — read this before wiring a delete button.** Deleting a Tag silently cascades: every assignment to every Person, Organization, and Deal is removed atomically. There is no preview, no confirmation step from the API, and no undo. A tag deleted by mistake must be recreated by hand and re-attached to every record that used to carry it — and the API does not preserve the prior assignment list anywhere you can read it back.
>
> Build a two-step confirmation in your UI. List the affected hosts (`GET /persons/?tags={id}`, `GET /organizations/?tags={id}`, `GET /deals/?tags={id}`) and surface the counts before issuing the DELETE.

**Response** `204 No Content`.

**Error** `403 Forbidden` — missing `Ibl.CRM/Tags/delete`.

**Error** `404 Not Found`.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/tags/7/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---

#### POST `/{host}/{id}/tags/` — attach a tag

Attach an existing tag to a Person, Organization, or Deal. The contract is identical across host types; the only thing that changes is the host path prefix.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | UUID (Person, Organization) or integer (Deal) | Host record id |

**Request body**

| Field | Type | Required | Description |
|---|---|---|---|
| `tag_id` | integer | Yes | Tag id to attach. Must belong to your Platform. Must be `≥ 1` |

```json
{"tag_id": 7}
```

**Response** `201 Created`

```json
{
  "assignment_id": 532,
  "tag": {"id": 7, "name": "Enterprise", "color": "#3F6BFF"}
}
```

**Response** `409 Conflict` — the tag is already attached to this host. Returns the existing `assignment_id` so you can treat the attempt as idempotent without an extra round-trip:

```json
{
  "detail": "Tag already attached.",
  "assignment_id": 488,
  "tag": {"id": 7, "name": "Enterprise", "color": "#3F6BFF"}
}
```

**Error** `400 Bad Request`

```json
{"tag_id": ["This field is required."]}
```

```json
{"tag_id": ["Ensure this value is greater than or equal to 1."]}
```

**Error** `403 Forbidden` — missing `Ibl.CRM/Tags/write`.

**Error** `404 Not Found` — host id unknown, host owned by a different Platform, or the tag doesn't belong to the same Platform as the host:

```json
{"detail": "Tag not found in this platform."}
```

```bash
curl -X POST "https://platform.iblai.app/api/crm/deals/184/tags/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tag_id": 7}'
```

```bash
curl -X POST "https://platform.iblai.app/api/crm/persons/c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80/tags/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tag_id": 7}'
```

```bash
curl -X POST "https://platform.iblai.app/api/crm/organizations/b2a1e7d3-6f4c-4b2a-8e9d-1c3a5b7d9f10/tags/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tag_id": 7}'
```

---

#### DELETE `/{host}/{id}/tags/{tag_id}/` — detach a tag

Detach a tag from a host record. Removes the assignment row only — the Tag itself is untouched.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `id` | UUID (Person, Organization) or integer (Deal) | Host record id |
| `tag_id` | integer | Tag id to detach |

**Response** `204 No Content`.

**Error** `403 Forbidden` — missing `Ibl.CRM/Tags/write`.

**Error** `404 Not Found` — host unknown, or the tag was not attached to this host:

```json
{"detail": "Tag not attached to this record."}
```

Detach is **not** idempotent — a second DELETE for the same tag returns `404`. UIs that retry on flaky networks should treat `404` on detach as a successful no-op rather than a hard error.

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/deals/184/tags/7/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/persons/c4d2b1a8-7c3e-4f9a-9d6b-9a2c4f1e7b80/tags/7/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

```bash
curl -X DELETE "https://platform.iblai.app/api/crm/organizations/b2a1e7d3-6f4c-4b2a-8e9d-1c3a5b7d9f10/tags/7/" \
  -H "Authorization: Token YOUR_ACCESS_TOKEN"
```

---
## 8. Person Onboarding (Link, Invite, Merge)

People in the CRM start their life as a person record — a row that holds emails, phone numbers, lifecycle stage, ownership, and tags, but is not yet a real account on your Platform. They become first-class Platform users in one of three ways: by **linking** to a user that already exists, by being **invited** via email, or by being **merged** into another person record when duplicates show up. This section walks through each path, the auto-link signal that ties them together, and the state transitions clients must expect.

### Decision tree

Use this to pick the right endpoint before you call anything:

```mermaid
flowchart TD
    Q{Is there already a Platform user for this person?} -->|Yes, user_id known| L[POST /persons/&#123;id&#125;/link-user/]
    Q -->|No, but the person has an email| I[POST /persons/&#123;id&#125;/invite/]
    Q -->|Duplicate person rows| M[POST /persons/merge/]
```

The three branches are not interchangeable. `link-user` requires a Platform user that already exists; `invite` requires a `primary_email` and produces a Platform user only once the invitee accepts; `merge` is a cleanup operation that re-parents related records and never touches Platform users.

---

### 8.1 Link an existing Platform user

Use this when you already know the Platform user the person corresponds to — for example, you imported a person list and then matched it to your existing user directory.

**Endpoint**

```
POST /api/crm/persons/{id}/link-user/
```

**Request**

```bash
curl -X POST \
  https://platform.iblai.app/api/crm/persons/8f0a3c1e-1d52-4a8b-9c2f-1f6e9b2c4d11/link-user/ \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"user_id": 4217}'
```

**200 OK** — returns the full person record:

```json
{
  "id": "8f0a3c1e-1d52-4a8b-9c2f-1f6e9b2c4d11",
  "platform": 1,
  "name": "Alice Chen",
  "primary_email": "alice@example.com",
  "platform_user": 4217,
  "active": false,
  "lifecycle_stage": "customer",
  "tags": [],
  "metadata": {},
  "created_at": "2026-05-30T14:02:11Z",
  "updated_at": "2026-06-04T09:18:44Z"
}
```

Note that `active` flips to `false` on a successful link — that is by design. Once a person is bound to a real Platform user, the user record is the source of truth and the person row becomes the historical sales context.

**Silent-refusal callout**

The link service refuses to rebind a person that is already bound to a *different* Platform user. There is no error response — the call returns **200 OK** with the existing binding untouched. Clients **MUST** verify the response body:

```js
if (response.platform_user !== requestedUserId) {
  // Refusal: the person was already linked to someone else.
  // The existing binding is preserved and no audit row is lost.
}
```

This is deliberate — silent rebinds would erase a prior link without trace. If your UI offered a "Link to user" action and you got back a different `platform_user`, surface a "this person is already linked to user X" message rather than claiming success.

**Status codes**

| Status | When |
| --- | --- |
| 200 | Linked, or silent refusal (compare `platform_user` against your request) |
| 400 | Request body missing `user_id` or wrong type |
| 403 | User exists but is not an active member of this Platform |
| 404 | Person not found in this Platform, or `user_id` does not exist |

**RBAC**

Requires `Ibl.CRM/Persons/write`. The target user must additionally have an active membership in the same Platform — otherwise the call returns 403 with the message *"User has no active UserPlatformLink to this platform; issue an invitation instead."*

---

### 8.2 Invite by email

Use this when the person is not yet a Platform user. The invitation reuses the standard Platform invitation pipeline, and on acceptance a Platform user is created and the auto-link signal (see 8.4) binds the person automatically.

**Endpoint**

```
POST /api/crm/persons/{id}/invite/
```

**Request**

```bash
curl -X POST \
  https://platform.iblai.app/api/crm/persons/8f0a3c1e-1d52-4a8b-9c2f-1f6e9b2c4d11/invite/ \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "redirect_to": "https://app.example.com/dashboard",
    "is_admin": false,
    "is_staff": false
  }'
```

The `is_admin` and `is_staff` flags control what the invitee receives on acceptance — both default to `false`. `redirect_to` is the URL the invitee lands on after accepting; leave it out to use the Platform default. An optional `enrollment_config` object may also be passed through to auto-enroll the invitee in courses, programs, or pathways on acceptance.

**201 Created**

```json
{
  "person_id": "8f0a3c1e-1d52-4a8b-9c2f-1f6e9b2c4d11",
  "invitation_id": 9821,
  "invitation_email": "alice@example.com",
  "platform_key": "acme",
  "auto_accept": true,
  "active": true,
  "redirect_to": "https://app.example.com/dashboard",
  "created": "2026-06-04T09:24:01Z"
}
```

The invitation email is dispatched out-of-band; the response confirms the invitation row was created, not that the email landed.

**Status codes**

| Status | When |
| --- | --- |
| 201 | Invitation created and queued for delivery |
| 400 | Person has no `primary_email` — cannot invite |
| 403 | Missing `Ibl.CRM/Invite/action` permission |
| 404 | Person not found in this Platform |
| 409 | An active invitation already exists for this email — response body includes the pre-existing `invitation_id` |
| 422 | Person is already linked to a Platform user (`platform_user` is set) |

The **409** response is informational, not a hard error — the caller can treat it as "already done, here is the invitation you wanted":

```json
{
  "detail": "Active PlatformInvitation already exists for this email + platform.",
  "invitation_id": 9821,
  "person_id": "8f0a3c1e-1d52-4a8b-9c2f-1f6e9b2c4d11",
  "platform_key": "acme"
}
```

**RBAC**

Requires `Ibl.CRM/Invite/action` — and this is the important part: invite is governed by a **separate RBAC bucket** from person-write. A role that can fully edit and delete people but does not carry `Ibl.CRM/Invite/action` cannot send invitations. This is by design — invitations send email and grant Platform access, so the privilege is split out from generic record editing.

---

### 8.3 Merge duplicates

Duplicates appear when the same human shows up through two channels — a CSV import plus a webform submission, two integrations pointing at the same address, a marketing list collision. Merge re-parents related records onto a chosen primary and marks the rest inactive.

**Endpoint**

```
POST /api/crm/persons/merge/
```

**Request**

```bash
curl -X POST \
  https://platform.iblai.app/api/crm/persons/merge/ \
  -H "Authorization: Token YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "primary_id": "8f0a3c1e-1d52-4a8b-9c2f-1f6e9b2c4d11",
    "duplicate_ids": [
      "2b41fa07-77a4-4f0c-8a13-9d6f7e2c4b88",
      "c7e91002-3aa1-4d2e-bff4-1e0d8a4f9a23"
    ]
  }'
```

**200 OK**

```json
{
  "primary_id": "8f0a3c1e-1d52-4a8b-9c2f-1f6e9b2c4d11",
  "merged_ids": [
    "2b41fa07-77a4-4f0c-8a13-9d6f7e2c4b88",
    "c7e91002-3aa1-4d2e-bff4-1e0d8a4f9a23"
  ],
  "reparented": {
    "deals": 7,
    "activities": 23,
    "tags": 4
  }
}
```

**What re-parents**

Inside a single transaction, three relations are moved from each duplicate onto the primary:

- **Deals** — every deal that pointed at a duplicate now points at the primary.
- **Activities** — calls, meetings, tasks, notes attached to a duplicate are re-attached.
- **Tag assignments** — moved across, with one wrinkle: if the primary already carries the same tag, the duplicate's assignment is **dropped silently** (the `(tag, person)` unique constraint forbids stacking). The `tags` count in `reparented` reflects every assignment **touched** — both moves and drops — not just successful moves. If you need to know exactly how many tags landed on the primary, compare before/after listings rather than reading the count.

**What happens to duplicates**

Duplicates are **not** deleted. Each duplicate's `active` flag is set to `false` and the row remains retrievable by id. This preserves audit trail and lets you investigate later — but it also means a `GET /persons/{duplicate_id}/` after a merge returns the inactive row, not a 404. Filter on `active=true` if your UI should hide them.

**Status codes**

| Status | When |
| --- | --- |
| 200 | Merge complete (or no-op rerun) |
| 400 | `primary_id` appears in `duplicate_ids`, or a duplicate belongs to another Platform, or `duplicate_ids` is empty |
| 403 | Missing `Ibl.CRM/Persons/write` |
| 404 | Primary not found in this Platform |

**RBAC**

Requires `Ibl.CRM/Persons/write`. Cross-Platform merges are explicitly forbidden — every id must resolve to a person on the caller's Platform.

---

### 8.4 Auto-Link Flow

The three actions above are explicit calls a CRM operator makes. There is also a fourth, implicit path: when a Platform user is created — through signup, invitation acceptance, or admin action — the system asynchronously walks the CRM and binds any matching person records automatically. This lets you bulk-import persons before invitations are sent; matching records bind automatically as the invitees register.

**How matching works**

After a Platform user is created with an active membership for a Platform, the system searches that Platform for person rows where:

- `primary_email` equals the new user's email (case-insensitive), **or** `platform_user` is already set to that user,
- `active` is `true`,
- the person belongs to a Platform the user is an active member of.

For every match, the link service runs: `platform_user` is set, `active` is flipped to `false`, and a `CRM_PERSON_LINKED_TO_USER` notification is dispatched for the configured recipients.

The match is gated by Platform membership — a new user with email `alice@example.com` only auto-links to person rows on Platforms they actually belong to. Cross-Platform leakage is not possible through this path.

**Sequence**

```mermaid
sequenceDiagram
    participant App as Your application
    participant API as CRM API
    participant Worker as Background worker
    participant Mailbox as Recipients

    App->>API: POST /persons/ with primary_email alice@x.com
    API-->>App: 201 Person (active true, platform_user null)
    Note over Worker: Later — alice@x.com signs up as a Platform user
    Worker->>API: Match persons by primary_email
    API->>API: Set platform_user, flip active to false
    API->>Mailbox: CRM_PERSON_LINKED_TO_USER notification
```

**State transition callout**

The auto-link runs in the background, so the response that created the Platform user returns **before** the link completes. Two consequences for clients:

- A `GET /persons/{id}/` issued moments after a signup may still show `active: true` and `platform_user: null`. A retry seconds later will show the linked state. Build polling or websocket-driven refresh into any UI that surfaces this.
- A person can flip from `active: true` to `active: false` between page loads, with no operator action in between. UIs that render a list of "active people" must tolerate rows disappearing on the next fetch; never crash on the transition.

**Edge cases**

- A user with no email is **skipped** — the system will not bulk-link every blank-`primary_email` person to a fresh emailless user.
- A user with no active Platform memberships is **skipped** — no Platforms to scan.
- A person already bound to a different Platform user is **left alone** (the silent-refusal rule from 8.1 applies here too).
- Re-running auto-link for the same user is a no-op once everything is bound; it is safe to retry on background-task failures.

---
## 9. Deal Lifecycle

A Deal moves through the stages of its Pipeline and reports a high-level `status` of `open`, `won`, or `lost`. Both `status` and `closed_at` are derived from the stage the Deal currently sits in — they are never written directly. Anything that needs to change where a Deal lives in the pipeline goes through one of three action endpoints, and each one writes an audit row and fires a notification so the timeline and the inbox stay in sync with the kanban.

This section is the contract for that flow: which endpoints to call, which fields are off-limits to PATCH, what the audit row looks like, what happens under concurrency, and what counts as a no-op.

### 9.1 The three transitions

There are exactly three endpoints that change a Deal's stage. Everything else — list, retrieve, create, partial update of metadata — leaves the stage alone.

- `POST /api/crm/deals/{id}/move-stage/`
  - Body: `{"stage_id": <int>}` OR `{"stage_code": "<slug>"}`
  - Moves the Deal to any stage in its Pipeline (initial, intermediate, or terminal).
  - Use this for kanban drag-and-drop and for explicit stage pickers.
- `POST /api/crm/deals/{id}/won/`
  - Body (optional): `{"stage_code": "<slug>"}`
  - Moves the Deal to a terminal won stage. If `stage_code` is omitted, the API resolves the first stage in the Pipeline with `is_won: true`, ordered by `sort_order`.
  - Use this for the "Mark as Won" button on a Deal detail page.
- `POST /api/crm/deals/{id}/lost/`
  - Body: `{"lost_reason": "<string>", "stage_code": "<slug>?"}`
  - `lost_reason` is REQUIRED — the API returns 400 if it is missing or blank. `stage_code` is optional and falls back to the first terminal lost stage by `sort_order`.
  - Use this for the "Mark as Lost" modal, which should always collect a reason before submitting.

All three return the updated Deal on success and the standard error envelope on failure.

### 9.2 Read-only fields

`status` and `closed_at` are server-managed and cannot be set by the client.

- A PATCH or PUT against `/api/crm/deals/{id}/` that includes `status` or `closed_at` returns `400` with a field-level error message naming the offending field.
- The fix is to call one of the three action endpoints above. Do not surface `status` or `closed_at` in a Deal edit form; they belong on the transition controls.

`stage` is also not directly writable on update — to change the stage, call `move-stage/`, `won/`, or `lost/`. Direct stage assignment is reserved for Deal creation.

### 9.3 Audit trail

Every successful stage move that actually changes the Deal's stage writes a system Activity. No-ops (Section 9.6) do not.

The audit Activity is shaped like this:

- `type`: `note`
- `is_done`: `true`, with `done_at` stamped to the transition time
- `title`: `"Stage changed"`
- `comment`: `"<From Display Name> → <To Display Name>"` — uses the human-readable stage names, not slugs (for example, `"Negotiation → Closed Won"`, not `"negotiation → closed-won"`).
- `owner`: the user who triggered the transition (the authenticated caller; null for service-account callers without an underlying user).
- `deal`: the Deal that moved.
- `person`: the Deal's `person`, so the audit row also surfaces in person-centric timelines.

Listing activities filtered by `?deal={id}` returns these system rows interleaved with user-logged Activities (calls, meetings, tasks, notes). To make the timeline readable:

- Render system stage-change rows with a distinct icon (an arrow or pipeline glyph works well) and a muted background.
- Do not show edit or "mark done" controls on them — they are already `is_done: true` and they describe an event, not a task.
- Keep them in chronological order with the rest of the timeline so reps can read the full story of the Deal in one pass.

### 9.4 Notification

Every successful, non-no-op stage move fires a `CRM_DEAL_STAGE_CHANGED` notification. The notification carries the Deal, the from-stage, the to-stage, and the user who triggered the move. Routing follows the standard CRM `RecipientsConfig` rules — owner, configured admins, or a custom list per Platform.

See Section 12 for the full context payload, the available channels, and how to wire recipient routing per Platform.

### 9.5 Concurrency

Stage moves on the same Deal are serialized.

When the API processes any of the three transition endpoints, it acquires a row lock on the Deal before reading its current stage, writing the new one, recording the audit row, and firing the notification. If two requests land at the same time — a kanban drag racing with someone clicking the Won button, or two reps acting on the same Deal — one wins and runs to completion, then the other proceeds against the now-updated state.

The practical consequences:

- Exactly one stage transition is recorded per logical move. You never get two audit rows for what was conceptually one drag.
- Exactly one `CRM_DEAL_STAGE_CHANGED` notification is fired per logical move. Recipients do not get duplicates.
- The losing request still gets a successful response (or a no-op response, if the winner happened to move the Deal to the same stage the loser was targeting).

The UI does not need to coordinate clients. The lock does the work. What the UI should do is refresh the Deal after a successful transition so the displayed stage matches reality, since another user's action may have run in between.

### 9.6 Idempotency

The three transitions are idempotent with respect to the Deal's current stage. Calling them when the Deal is already where you are asking it to go is a no-op:

- `move-stage/` to the current stage: no write, no audit row, no signal, no notification. Returns `200` with the unchanged Deal.
- `won/` on a Deal already in its won stage: no-op. Returns `200`.
- `lost/` on a Deal already in its lost stage: the stage transition is a no-op (no audit row, no signal, no notification), **but `lost_reason` is overwritten unconditionally with whatever value is in the request body**. Pass the original reason on retries, or PATCH the Deal directly if you want to change only the reason without re-firing the lost action.

This means clients can safely retry transition requests on transient network failures without worrying about double-recording the move.

### 9.7 Re-opening a Deal

Won and lost are not one-way doors. Moving a Deal in a terminal stage back to a non-terminal stage re-opens it:

- `status` flips back to `open`.
- `closed_at` is cleared.
- An audit Activity is still written (`"Closed Won → Negotiation"`, for example).
- A `CRM_DEAL_STAGE_CHANGED` notification still fires, so the owner and configured admins know the Deal is live again.

Use `move-stage/` to re-open. There is no dedicated "reopen" endpoint — the stage move IS the reopen, and the derived `status` follows the destination stage's `is_won` / `is_lost` flags.

### 9.8 Diagram — winning a deal

```mermaid
sequenceDiagram
    participant C as Client
    participant API as CRM API
    participant Recipients as Notification recipients
    C->>API: POST /deals/&#123;id&#125;/won/
    API->>API: Resolve target stage (first is_won)
    API->>API: Lock deal row
    API->>API: Update stage + status + closed_at
    API->>API: Write audit Activity ("Stage changed: Negotiation → Closed Won")
    API->>Recipients: CRM_DEAL_STAGE_CHANGED notification
    API-->>C: 200 Deal (status: won, closed_at set)
```

---
## 10. Activity Timeline & Auto-Records

Activities are the chronological feed for a deal or a person. Calls, meetings, notes, tasks, emails — anything you want logged ends up here. Some rows are user-authored (a sales rep writing a call note), others are written by the server itself when a deal moves through the pipeline. The same endpoint serves both, and the client is responsible for telling them apart.

### 10.1 Reading a timeline

The list endpoint is filterable by either parent. For a deal-centric feed:

```bash
curl -X GET "https://platform.iblai.app/api/crm/activities/?deal=314" \
  -H "Authorization: Token $TOKEN"
```

For a person-centric feed:

```bash
curl -X GET "https://platform.iblai.app/api/crm/activities/?person=9c6f4a2e-1b88-4a0e-9b71-2c2f7a1d6e44" \
  -H "Authorization: Token $TOKEN"
```

Both calls return the standard paginated envelope. Default order is newest-first by `created_at`.

```json
{
  "count": 1,
  "next_page": null,
  "previous_page": null,
  "results": [
    {
      "id": 8821,
      "platform": 1,
      "type": "call",
      "title": "Discovery call with procurement lead",
      "location": "Zoom",
      "comment": "Walked through pricing tiers and the pilot timeline.",
      "deal": 314,
      "person": "9c6f4a2e-1b88-4a0e-9b71-2c2f7a1d6e44",
      "owner": 42,
      "schedule_from": "2026-06-04T15:00:00Z",
      "schedule_to": "2026-06-04T15:30:00Z",
      "reminder_at": "2026-06-04T14:45:00Z",
      "reminder_sent": false,
      "is_done": true,
      "done_at": "2026-06-04T15:33:11Z",
      "metadata": {"call_outcome": "positive"},
      "created_at": "2026-06-03T09:12:00Z",
      "updated_at": "2026-06-04T15:33:11Z"
    }
  ]
}
```

You can combine filters — `?deal=314&type=call&is_done=false` to render an upcoming-calls panel on a deal page, for example.

### 10.2 Distinguishing system rows from user rows

The CRM writes its own Activities when a deal transitions between stages (see Section 9). These appear in the same feed as user-authored entries, but you usually want to render them with a distinct affordance — a small system icon, no edit button, no mark-done control. There is no dedicated `source` field on the model; distinguish them client-side with a simple predicate:

```ts
const isStageChangeAudit = (a: Activity): boolean =>
  a.type === 'note' && a.title === 'Stage changed';
```

System rows arrive already completed (`is_done: true`, `done_at` set to the transition timestamp). Do not offer edit or done controls for them — the user did not author the row and there is nothing to mark done.

### 10.3 Marking an activity done

```bash
curl -X POST "https://platform.iblai.app/api/crm/activities/8821/done/" \
  -H "Authorization: Token $TOKEN"
```

No request body. The action is idempotent: the first call flips `is_done` to `true` and stamps `done_at` with the current server time. Subsequent calls return the same activity with the ORIGINAL `done_at` preserved — the server does not re-stamp on repeat invocations. This matters for offline-capable clients that may retry the same request after a network blip; you will not see the completion time drift forward each time the queue flushes.

The response is the full updated Activity object, so the client can replace the row in its local store without a follow-up GET.

### 10.4 Schedule semantics

The two scheduling fields combine to express three different intents. Pick the shape that matches what you are recording — do not invent dates to satisfy both fields.

| `schedule_from` | `schedule_to` | Read as |
|---|---|---|
| null | null | A past log entry — what happened, recorded after the fact |
| set | null | A scheduled task with a start time but no fixed end |
| set | set | A meeting or time-bounded event |

A call note written ten minutes after the call ended is the first shape. A "follow up next Tuesday" task is the second. A 30-minute demo on the calendar is the third. The server does not enforce a relationship between these fields beyond storage, so a malformed combination (for example, `schedule_to` before `schedule_from`) is the client's responsibility to prevent.

### 10.5 Reminders

The `reminder_at` field is set by the caller — typically a fixed offset before `schedule_from` (15 minutes before is a common default for meetings). The `reminder_sent` flag is server-managed; do not write to it from the client.

> **Reminder delivery is not currently dispatched server-side.** Set `reminder_at` to track intent and surface a local in-app prompt; the field round-trips correctly. `reminder_sent` will remain `false` until a server-side dispatcher is wired.

### 10.6 Attaching an activity

Every activity must attach to a deal OR a person — or both. Posting an Activity with neither field set returns:

```json
{
  "detail": ["Activity must attach to a `deal` or a `person`."]
}
```

with a 400 status. The serializer raises this before any database write, so a failed create has no side effects.

If both `deal` and `person` are set, the person must be the same person already attached to the deal. A mismatch returns a 400 with a field-level error explaining the constraint — see Section 7.6 for the canonical attachment rules and the rationale for why we enforce this at write time rather than letting the timeline display orphaned rows.

A typical create call for a meeting attached to both:

```bash
curl -X POST "https://platform.iblai.app/api/crm/activities/" \
  -H "Authorization: Token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "meeting",
    "title": "Pricing review",
    "deal": 314,
    "person": "9c6f4a2e-1b88-4a0e-9b71-2c2f7a1d6e44",
    "schedule_from": "2026-06-10T16:00:00Z",
    "schedule_to": "2026-06-10T16:45:00Z",
    "reminder_at": "2026-06-10T15:45:00Z"
  }'
```

The response is the created Activity, ready to splice into the timeline view without a re-fetch.

---
## 11. Tagging

Tags are coloured labels you can attach to people, organizations, and deals. Use them for ad-hoc segmentation — VIP, trial, churn-risk, newsletter-2026, whatever the sales team needs this quarter — without inventing custom fields or migrating schema. A tag is just a `(name, color)` pair scoped to your Platform; attaching it to a host is a cheap join row, and detaching it leaves the host untouched.

### 11.1 Tag CRUD

Full reference for the tag resource itself is in Section 7.7. The short version:

```bash
curl -X POST https://platform.iblai.app/api/crm/tags/ \
  -H "Authorization: Token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "VIP", "color": "#3F6BFF"}'
```

Two rules to know up front:

- **Names are unique per Platform.** Comparison is case-sensitive and whitespace is trimmed before the uniqueness check, so `"VIP"` and `" VIP "` collide but `"VIP"` and `"vip"` do not.
- **Colors must match `^#[0-9a-fA-F]{6}$`.** Three-digit shorthand (`#abc`) is rejected. If you omit `color`, the server stores `#888888`.

Violations come back as a 400 with the usual field-error envelope; let the user fix the input and retry.

### 11.2 Attach a tag to a host

To attach an existing tag to a person, organization, or deal:

```
POST /api/crm/{host}/{id}/tags/
```

`{host}` is one of `persons`, `organizations`, `deals`. The body is the tag ID you want to attach:

```bash
curl -X POST https://platform.iblai.app/api/crm/persons/42/tags/ \
  -H "Authorization: Token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tag_id": 7}'
```

On success you get `201 Created` with the new assignment plus the embedded tag, so the client never needs a second round trip to render the chip:

```json
{
  "assignment_id": 318,
  "tag": {
    "id": 7,
    "name": "VIP",
    "color": "#3F6BFF"
  }
}
```

Two edge cases worth wiring into the UI:

- **Already attached.** The server returns `409 Conflict` with the exact same shape, where `assignment_id` is the *existing* row. Treat 409 as a no-op success — the user clicked twice or two tabs raced; either way the desired state is already in place. Do not show a red error toast for this case.
- **Tag from another Platform.** The server returns `404 Not Found`. Existence across Platforms is never leaked, so a 404 here means either the tag does not exist or it does not belong to you. Same UX: "Tag not found."

### 11.3 Detach a tag from a host

```
DELETE /api/crm/{host}/{id}/tags/{tag_id}/
```

```bash
curl -X DELETE https://platform.iblai.app/api/crm/persons/42/tags/7/ \
  -H "Authorization: Token $TOKEN"
```

Returns `204 No Content` on success with no body. Returns `404 Not Found` (`{"detail": "Tag not attached to this record."}`) if the tag was never attached to that host (or the tag/host does not exist or is not on your Platform — again, no existence leak across Platforms).

**Detach is not idempotent on the server** — the second `DELETE` against an already-detached pair returns `404`, not `204`. From the user's point of view, however, both responses mean the tag is gone, so client code can treat 204 and 404 as the same success state when reconciling local state.

### 11.4 Reading tags off a host

Person, Organization, and Deal serializers expose a read-only `tags` array on every list and detail response. You do not need a separate call to render tag chips on a card or row:

```json
"tags": [
  {"id": 7, "name": "VIP", "color": "#3F6BFF"},
  {"id": 12, "name": "trial", "color": "#22AA66"}
]
```

The array is always present on the response. An empty array means the host has no tags attached — it never comes back as `null` or missing, so your renderer can iterate unconditionally.

### 11.5 Filtering by tag

All three host list endpoints accept a `tags` filter with **OR** semantics. Two equivalent forms are supported:

```
GET /api/crm/persons/?tags=7&tags=12
GET /api/crm/persons/?tags=7,12
```

Both queries return every person tagged with `7` *or* `12`. A person carrying both tags appears once — the backend de-duplicates, so you do not need a `DISTINCT` step on your end.

The `tags` filter composes with every other filter on the list endpoint. For example, "VIP people in the `customer` lifecycle stage owned by user 4":

```
GET /api/crm/persons/?tags=7&lifecycle_stage=customer&owner=4
```

Responses use the standard pagination envelope `{count, next_page, previous_page, results}` where `count`, `next_page`, and `previous_page` are integers (`next_page` / `previous_page` are `null` at the ends).

### 11.6 Permission callout

Attach and detach are gated by a **separate RBAC bucket** from the host itself. The action checks `Ibl.CRM/Tags/write` on the calling role — *not* `Ibl.CRM/Persons/write`, `Ibl.CRM/Organizations/write`, or `Ibl.CRM/Deals/write`.

In practice this means a role can be allowed to edit people but forbidden from tagging them, or the other way around. When deciding whether to render the "Add tag" affordance on a person/organization/deal card, check the tag-write permission independently of the host-write permission. The most common UX bug here is hiding the tag chooser behind the host's edit gate — which silently locks tag-write users out of a feature they actually have.

### 11.7 Destructive cascade callout

> **Deleting a tag silently removes it from every person, organization, and deal it is attached to.** There is no preview endpoint that tells you how many hosts will lose the tag, and there is no undo. The assignment rows are hard-deleted along with the tag itself.

Because the blast radius is invisible from the API, the UI has to carry the warning. Gate `DELETE /api/crm/tags/{id}/` behind a strong confirm dialog with copy along these lines:

> *"This will remove the tag from every person, organization, and deal it is attached to. This cannot be undone."*

If you want a softer migration path for your users, build a client-side "archive" convention — e.g. rename the tag to `zz_archived_VIP` and recolor it grey — and reserve true delete for tags you are certain nobody is filtering on.

---
## 12. Notifications

The CRM fires three notification types. Each is configurable per Platform — you can change recipients, edit the email/push template content, or toggle the type off entirely through the notification templates API. The mechanics (template storage, channel routing, preference resolution) are the same machinery every other notification type in the system uses; cross-reference the notification system documentation for the full template payload and lifecycle. This section covers only what is CRM-specific: when each type fires, the context keys the template can interpolate, and how recipient routing works.

### 12.1 The three types

| Type | Fires when | Context provided to template | Default recipients |
|---|---|---|---|
| `CRM_PERSON_CREATED` | A person is created (API, admin, or import) | `person_id`, `person_name`, `person_email`, `person_lifecycle_stage`, `person_job_title`, `owner_username` | Platform admins + person owner |
| `CRM_DEAL_STAGE_CHANGED` | A deal moves between stages (`move-stage` / `won` / `lost`). No-op transitions (same stage to same stage) are suppressed. | `deal_id`, `deal_title`, `deal_status`, `deal_lead_value`, `deal_currency`, `from_stage_code`, `from_stage_name`, `to_stage_code`, `to_stage_name`, `person_name`, `actor_username` | Platform admins + deal owner |
| `CRM_PERSON_LINKED_TO_USER` | A person is bound to a Platform user — either by the auto-link signal that matches on email when a new user registers, or by an explicit call to `/link-user/` | `person_id`, `person_name`, `person_email`, `linked_user_id`, `linked_user_username`, `linked_user_email` | Platform admins + person owner |

All three are dispatched asynchronously after the writing transaction commits. If the write that triggered the signal is rolled back — a validation failure further down the request, a `move-stage` call that raises mid-transition — no notification is produced. The signal is observed, but the dispatch only fires on the post-commit hook, so consumers never see ghost notifications for state that never persisted.

Context keys are populated at signal time from the live object. A template that references `{{ deal_lead_value }}` reads the value as it stood the moment the stage transition committed; later edits to the deal do not re-render or re-send the notification.

### 12.2 Recipient modes

Every CRM notification template — and in fact any notification type that opts into the shared recipients pipeline — can be configured per Platform with one of five recipient modes:

| Mode | Effect |
|---|---|
| `platform_admins_only` | Deliver only to active Platform admins. The object's owner is ignored. |
| `object_owner_only` | Deliver to the object's owner. If the owner field is unset, fall back to Platform admins so the notification is not silently dropped. |
| `object_owner_only_strict` | Deliver to the object's owner only. If the owner is unset, no one is notified. Use when the notification is meaningless without an owner. |
| `platform_admins_and_object_owner` | Default. Both audiences receive the notification, deduplicated so an admin who is also the owner does not get two copies. |
| `custom` | Deliver to a hand-picked list of users, user groups, or RBAC role-policy holders. Admins and the owner are not implicitly included. |

"Object owner" refers to the `owner` field on the triggering object:

- For `CRM_PERSON_CREATED` and `CRM_PERSON_LINKED_TO_USER`, that is `person.owner`.
- For `CRM_DEAL_STAGE_CHANGED`, that is `deal.owner`.

If an object is created without an owner — for instance, an imported person row that has no assigned account manager yet — the fallback behavior described in the table applies. There is no separate concept of an organization-level owner being used for routing.

### 12.3 Custom recipient shape

When the mode is `custom`, the template's `recipients_custom_recipients` field holds a list. Each entry takes one of three shapes:

```json
{"type": "user", "id": 123}
```

```json
{"type": "user_group", "id": 7}
```

```json
{"type": "rbac_policy", "policy_name": "CRM Manager"}
```

The three types compose freely — a single custom list can mix individual users, groups, and policy holders. The resolver expands each entry, unions the results, deduplicates, and then runs the active-Platform-membership filter described below.

Custom recipients are re-filtered against active Platform membership at delivery time. A stale entry — a user who has left the Platform, a group whose roster has changed, a policy that has been reassigned — simply contributes no one to that send. You do not need to prune the list manually when membership changes; the resolver does it on every dispatch. This means a single custom recipient list can be maintained centrally even if individual users come and go.

If every entry in a custom list resolves to zero active members, the notification produces no deliveries for that Platform on that event. It is not auto-promoted back to the default mode; "no recipients" is treated as a valid configured outcome.

### 12.4 Changing recipients

Recipient configuration lives on the notification template, not on the CRM models. To change who hears about, say, deal stage transitions on a given Platform, PATCH the template for `CRM_DEAL_STAGE_CHANGED` on that Platform via the notification templates API and set:

- `recipients_recipient_mode` — one of the five modes from 12.2.
- `recipients_custom_recipients` — required when the mode is `custom`; a list of target dicts as shown in 12.3. Ignored for the other modes.

The exact endpoint path, full payload schema, and authentication requirements are documented under the notification system. The CRM does not add a separate API surface for this — the same template management endpoints that handle every other notification type handle these three.

Toggling a type off entirely is also done at the template level via the standard enable/disable flag on the notification template. The CRM signal still fires; the callback short-circuits before resolving recipients or dispatching. There is no CRM-side setting to suppress notifications.

### 12.5 Channels

Email delivery is wired by default for all three CRM notifications. Whether a given recipient actually receives mail depends on their individual notification preferences, which are honored by the underlying delivery machinery — the CRM does not bypass user preferences.

Additional channels (in-app feed, push notification) follow the template's channel configuration if added there; the CRM callbacks themselves dispatch email. Consult the templates API to inspect or change the active channel set per type, the same way you change recipients.

### 12.6 Diagram — notification routing

```mermaid
flowchart LR
    E[CRM event - person created, deal stage change, person linked] --> S[Notification service - post-commit dispatch]
    S --> R{Recipient mode}
    R -->|platform_admins_only| A[Active Platform admins]
    R -->|object_owner_only / strict| O[Object owner - person.owner or deal.owner]
    R -->|platform_admins_and_object_owner default| AO[Admins + owner deduped]
    R -->|custom| C[Custom users / user groups / RBAC policy holders]
    A --> F[Active Platform membership filter]
    O --> F
    AO --> F
    C --> F
    F --> M[Email + push delivery, honoring per-user preferences]
```

The membership filter is the last gate before delivery. Every resolved recipient — regardless of mode — is checked against the Platform's active membership at dispatch time. A user who has been removed from the Platform between the event and the dispatch does not receive the notification, even if they appear in a custom list, were the recorded `owner`, or were an admin at the time the event fired.

---
## 13. Filtering & Pagination

Every CRM list endpoint speaks the same dialect: the same pagination envelope, the same date-range suffixes, the same foreign-key-by-ID convention, and the same tag query shape. Learn it once and the rest of the API reads itself.

### 13.1 Pagination envelope

Every list endpoint returns the same envelope:

```json
{
  "count": 142,
  "next_page": 2,
  "previous_page": null,
  "results": [ ... ]
}
```

`next_page` and `previous_page` are integer page numbers (or `null`), NOT URLs. Walk pages with `?page=N`:

```
GET /api/crm/persons/?page=2
GET /api/crm/persons/?page=3
```

When `next_page` is `null` you have reached the last page. When `previous_page` is `null` you are on page 1.

`page_size` is Platform-configurable; 100 is a common deployment cap. Clients should treat the returned page size as authoritative and not assume a fixed number of results per page.

### 13.2 Common query parameters

Three patterns recur across resources. Once you have internalized them, you can predict the filter surface of any new endpoint.

**Date ranges.** Any field ending in `__gte` / `__lte` takes an ISO 8601 date or datetime. Both bounds are inclusive.

```
GET /api/crm/persons/?created_at__gte=2026-06-01T00:00:00Z
GET /api/crm/deals/?expected_close_date__lte=2026-12-31
```

Mix-and-match to bracket a window:

```
GET /api/crm/activities/?schedule_from__gte=2026-06-01T00:00:00Z&schedule_from__lte=2026-06-07T23:59:59Z
```

**Foreign-key filters.** Pass the related record's primary identifier — never its display name. Numeric IDs for Users and Pipelines; UUIDs for Persons, Organizations, and Deals.

```
GET /api/crm/deals/?owner=42
GET /api/crm/persons/?organization=4f2a1b8e-...-9c3d
GET /api/crm/deals/?pipeline=1
```

**Tag filters.** Any host with tags accepts either repeated params or a CSV:

```
GET /api/crm/persons/?tags=7&tags=12
GET /api/crm/persons/?tags=7,12
```

Both forms use OR semantics — a Person tagged with either `7` or `12` matches. The response never duplicates rows even when a record matches multiple tag IDs.

### 13.3 Custom-metadata filtering

Every resource carries a free-form `metadata` JSON object. Person and Deal list endpoints support `?metadata__has_key=fieldName` to find records that have that custom key set, regardless of its value. Useful for staged migrations or external-system tracking.

```
GET /api/crm/persons/?metadata__has_key=salesforce_id
GET /api/crm/deals/?metadata__has_key=external_quote_ref
```

The match is strict key presence — `null` values still count as present, missing keys do not.

There is no operator for filtering by metadata value. If you need value-based segmentation, fetch the page and filter client-side, or promote the field to a first-class column.

### 13.4 Cookbook

A copy-paste set of common queries. Substitute `{...}` placeholders with real IDs from your session.

| Goal | Query |
|---|---|
| Qualified leads owned by me | `/api/crm/persons/?lifecycle_stage=qualified&owner={my_user_id}` |
| People in an Organization, created this month | `/api/crm/persons/?organization={uuid}&created_at__gte=2026-06-01T00:00:00Z` |
| People with a custom metadata key | `/api/crm/persons/?metadata__has_key=salesforce_id` |
| Organizations by name typeahead | `/api/crm/organizations/?name={q}` |
| Open deals in a pipeline | `/api/crm/deals/?pipeline={id}&status=open` |
| Deals expected to close this month | `/api/crm/deals/?expected_close_date__gte=2026-06-01&expected_close_date__lte=2026-06-30` |
| My open tasks this week | `/api/crm/activities/?type=task&is_done=false&owner={my_user_id}&schedule_from__gte={week_start}&schedule_from__lte={week_end}` |
| Anything tagged X OR Y | `/api/crm/persons/?tags=7&tags=12` |
| All activities on a deal, newest first | `/api/crm/activities/?deal={deal_id}` |
| Activities completed on a person | `/api/crm/activities/?person={uuid}&is_done=true` |

These compose freely — add `&page=2`, layer a date range on top of an owner filter, or stack a tag filter onto a lifecycle filter. The backend ANDs distinct parameters and ORs repeated values of the same parameter (the tag rule above).

### 13.5 Per-resource filter index

Quick lookup of every filter exposed, by resource. If a parameter is not listed here, the endpoint silently ignores it.

| Resource | Filters |
|---|---|
| Persons | `lifecycle_stage`, `owner`, `organization`, `created_at__gte`, `created_at__lte`, `metadata__has_key`, `tags` |
| Organizations | `name`, `owner`, `tags` |
| Pipelines | `code`, `name`, `is_default` |
| Stages (nested) | `code`, `is_won`, `is_lost` |
| Lead Sources | `code`, `name` |
| Deals | `status`, `pipeline`, `stage`, `owner`, `source`, `person`, `organization`, `expected_close_date__gte`, `expected_close_date__lte`, `created_at__gte`, `created_at__lte`, `metadata__has_key`, `tags` |
| Activities | `type`, `is_done`, `owner`, `deal`, `person`, `schedule_from__gte`, `schedule_from__lte`, `metadata__has_key` |
| Tags | `name`, `created_at__gte`, `created_at__lte` |

> **No global search.** There is no cross-resource search endpoint and no substring filter on `Person.name`. Combine the filters above and do final substring matching client-side if needed.

---
## 14. Roles & Permissions

The CRM ships four roles per Platform. Assign them through the standard Platform role-management surface. Roles are seeded automatically — you do not create them.

### 14.1 The four roles

| Role | Mandate |
|---|---|
| **CRM Viewer** | Read everything across the CRM. Write nothing. |
| **CRM User** | Day-to-day operator. Full create / update / delete on people, organizations, deals, activities, tags. Pipelines are read-only — pipeline topology is an admin job. Cannot send invitations. |
| **CRM Manager** | Wildcard access to every CRM action, including invitations and pipeline editing. |
| **CRM Inviter** | Narrow role: read people and send invitations only. Cannot edit or create people. |

### 14.2 Full permission matrix

| Resource (bucket) | Action codes | CRM Viewer | CRM User | CRM Manager | CRM Inviter |
|---|---|---|---|---|---|
| `Ibl.CRM/Persons` | `list`, `read` | ✓ | ✓ | ✓ | ✓ |
| `Ibl.CRM/Persons` | `action`, `write`, `delete` | — | ✓ | ✓ | — |
| `Ibl.CRM/Organizations` | `list`, `read` | ✓ | ✓ | ✓ | — |
| `Ibl.CRM/Organizations` | `action`, `write`, `delete` | — | ✓ | ✓ | — |
| `Ibl.CRM/Pipelines` | `list`, `read` | ✓ | ✓ | ✓ | — |
| `Ibl.CRM/Pipelines` | `action`, `write`, `delete` | — | — | ✓ | — |
| `Ibl.CRM/Deals` | `list`, `read` | ✓ | ✓ | ✓ | — |
| `Ibl.CRM/Deals` | `action`, `write`, `delete` | — | ✓ | ✓ | — |
| `Ibl.CRM/Activities` | `list`, `read` | ✓ | ✓ | ✓ | — |
| `Ibl.CRM/Activities` | `action`, `write`, `delete` | — | ✓ | ✓ | — |
| `Ibl.CRM/Tags` | `list`, `read` | ✓ | ✓ | ✓ | — |
| `Ibl.CRM/Tags` | `action`, `write`, `delete` | — | ✓ | ✓ | — |
| `Ibl.CRM/Invite` | `action` | — | — | ✓ | ✓ |

Notes:
- Lead source endpoints fall under the `Ibl.CRM/Pipelines` bucket — they are administrative.
- Stage CRUD also lives under `Ibl.CRM/Pipelines` — moving a stage is admin work; moving a deal between stages is a deal action.

### 14.3 Action verb mapping

HTTP verbs map onto the five canonical action codes as follows:

| HTTP | Action code |
|---|---|
| `GET` (list endpoint) | `list` |
| `GET` (detail endpoint) | `read` |
| `POST` (create) | `action` |
| `POST` (custom action: `move-stage`, `won`, `lost`, `done`, `link-user`, `merge`) | `action` |
| `POST` attach-tag, `DELETE` detach-tag | `write` (on `Ibl.CRM/Tags`) |
| `PATCH` / `PUT` | `write` |
| `DELETE` (resource delete) | `delete` |

### 14.4 Two permissions worth a second look

- **Invitation is its own bucket.** A role with `Ibl.CRM/Persons/write` does not have `Ibl.CRM/Invite/action`. Check the invitation right independently when deciding whether to expose an "Invite" affordance.
- **Tag attach/detach require `Ibl.CRM/Tags/write`.** A role with `Ibl.CRM/Persons/write` cannot tag people unless it also has tag write. This is intentional — it lets you delegate tag-graph mutation separately from person mutation.

---
## 15. Error Reference

The API returns standard HTTP status codes with JSON bodies. The table below covers every class of failure you should plan for so the frontend can route the user to the correct recovery path without guessing.

### 15.1 Status codes

| Status | Class | Common causes | What to do |
|---|---|---|---|
| 400 | Validation | Required field missing or invalid; enum out of range; blank or whitespace-only `name`; invalid tag color hex; attempt to write a server-managed field (`Deal.status`, `Deal.closed_at`, `Activity.done_at`); Activity attached to neither deal nor person; Activity's `person` does not match `Deal.person`; `lost` action missing `lost_reason` | Fix the field and retry. For deal status changes, use the `move-stage`, `won`, or `lost` action endpoints (Section 9). |
| 401 | Unauthorized | Missing or invalid token | Provide `Authorization: Token YOUR_ACCESS_TOKEN` |
| 403 | Permission denied | RBAC role does not grant the required action; service-account API key is bound to a different Platform than the resource | Verify the role assignment (Section 14). The invitation action is its own bucket — check `Ibl.CRM/Invite/action` independently from `Ibl.CRM/Persons/write`. |
| 404 | Not found | Resource does not exist OR exists on another Platform (existence is not leaked across Platforms) | Verify the ID and that the caller's Platform owns it. Do not differentiate "not found" from "no access" in user-facing copy. |
| 409 | Conflict | Tag already attached to the host (response carries the existing `assignment_id`); pending invitation already exists for the person (response carries existing `invitation_id`); pipeline or stage delete attempted while deals are attached | For tag/invitation conflicts, treat the existing assignment / invitation as the result. For pipeline / stage deletes, move or close attached deals first. |
| 422 | Unprocessable | Invitation attempted on a person already linked to a Platform user | Skip the invitation — the person already has an account |

### 15.2 Sample error bodies

**Validation (400) — direct write to a server-managed field**:

```json
{
  "status": "Service-managed — write via `POST /deals/{id}/move-stage/`, `won/`, or `lost/`."
}
```

**Validation (400) — Activity with no parent**:

```json
{
  "detail": ["Activity must attach to a `deal` or a `person`."]
}
```

**Conflict (409) — tag already attached**:

```json
{
  "detail": "Tag already attached.",
  "assignment_id": 491,
  "tag": {"id": 7, "name": "VIP", "color": "#3F6BFF"}
}
```

**Conflict (409) — pending invitation**:

```json
{
  "detail": "Active PlatformInvitation already exists for this email + platform.",
  "invitation_id": 8821,
  "person_id": "8f1c2d4e-7a93-4b21-9e1f-2a6c8d4f5b10",
  "platform_key": "acme-learning"
}
```

**Unprocessable (422) — person already linked**:

```json
{
  "detail": "Person already linked to a platform user."
}
```

### 15.3 Cross-Platform existence is not leaked

A 404 from any endpoint means *either* the resource does not exist *or* it exists on another Platform. Treat the two cases identically — do not surface a distinction in your UI. This is a deliberate design choice to prevent enumeration attacks across Platforms.

---
## 16. Best Practices

Ground-tested advice for building reliable integrations on the CRM API. Each item is short, opinionated, and links back to the section that explains the underlying mechanic. Treat this as a pre-flight checklist before you ship.

### 16.1 Inspect the default seed before creating pipelines

Every Platform is seeded with one default pipeline and a set of default lead sources at creation time. The pipeline contains six stages with the codes `new`, `qualified`, `proposal`, `negotiation`, `won`, and `lost`. The lead sources are `web`, `referral`, `cold_call`, and `advertisement`.

Before your onboarding flow offers a "Create pipeline" button or seeds its own lead sources, call:

- `GET /api/crm/pipelines/`
- `GET /api/crm/lead-sources/`

If the seed is present, surface it as the working pipeline and skip creation entirely. Re-seeding produces duplicate records that confuse later filters. See Sections 7.3 and 7.4 for the full seed contract.

### 16.2 Reference stages by `code`, not by display name

Stage `name` values are editable by Platform admins — a stage labeled "Qualified" today may be "Discovery" tomorrow. The `code` field is stable and is what the action endpoints, terminal-stage resolution, and the auto-Activity audit row all key off internally.

Any code path that branches on which stage a deal occupies — kanban column placement, conditional UI, analytics rollups — should read `stage.code`, not `stage.name`. Reserve `name` for rendering. See Section 7.3.

### 16.3 Use the action endpoints for deal status changes

The deal serializer rejects direct writes to `status` and `closed_at` with a 400. Status transitions only happen through:

- `POST /api/crm/deals/{id}/move-stage/`
- `POST /api/crm/deals/{id}/won/`
- `POST /api/crm/deals/{id}/lost/`

Each of these writes the audit Activity row, fires the deal-stage-changed notification, and (for `won` / `lost`) sets `closed_at` atomically. PATCHing the fields directly would skip every one of those side effects and leave the timeline silently incomplete. See Section 7.5 and Section 9.

### 16.4 Always provide a `lost_reason`

`POST /deals/{id}/lost/` requires a non-empty `lost_reason` in the body and returns 400 if it is missing. Build the field into your UI before the user can submit the action — a free-text textarea with a short required-field hint is the minimum viable shape.

Do not infer or auto-fill a reason on the user's behalf. Lost reasons drive pipeline analytics; a synthetic value pollutes the data permanently. See `POST /deals/{id}/lost/` in Section 7.5.

### 16.5 Render system audit Activities differently

When a deal transitions stages, the move-stage action writes a system Activity with `type === 'note'` and `title === 'Stage changed'`. These appear in the same `/api/crm/activities/` list as user-authored notes, calls, and tasks.

In your timeline UI:

- Render them with a distinct icon (a system / arrow glyph works well)
- Suppress edit, delete, and mark-done controls — they are already complete and immutable in intent
- Keep them visually quieter than user-authored entries so the timeline still reads as a human history

See Section 7.6 and Section 10.

### 16.6 Verify the response of `link-user`

`POST /api/crm/persons/{id}/link-user/` returns 200 even when the link is silently refused — for example, when the person is already bound to a different Platform user. The endpoint does not raise; it just declines to overwrite.

Compare `response.platform_user` against the `user_id` you sent in the request body. A mismatch means the link did not take and manual reconciliation is required. Surface that case as an explicit warning in your UI, not a success toast. See `POST /persons/{id}/link-user/` in Section 7.1 and Section 8.1.

### 16.7 Confirm before deleting a tag

Tag deletion cascades silently and completely: every assignment of that tag on every host type — people, organizations, and deals — is removed in a single transaction. There is no dry-run preview endpoint, no soft-delete, and no undo.

Use a confirmation modal that:

- Names the tag explicitly ("Delete tag *Enterprise*?")
- States the consequence plainly ("This will remove the tag from N records across people, organizations, and deals.")
- Requires an affirmative click, not just a single OK button

See Section 7.7 and Section 11.

### 16.8 Treat 404 as "not visible to you"

A 404 from any CRM endpoint may mean the resource genuinely does not exist *or* that it exists on another Platform and the resolved Platform scope hides it from your token. The API intentionally does not distinguish the two — leaking existence across Platforms would itself be a permission violation.

Do not differentiate in user-facing copy. A single "Not found or not accessible" message is the correct response. See Section 3 and Section 15.3.

### 16.9 Paginate every list call

Every list endpoint returns the envelope `{count, next_page, previous_page, results}`. `count` is the total matching records. `next_page` and `previous_page` are integer page numbers (or `null` at the boundaries) — not URLs.

Walk pages with `?page=N`. Do not assume the first page holds the full result set: `page_size` is Platform-configurable and the default may be smaller than your data. Build pagination into list views from day one rather than retrofitting it after a customer complains about missing records. See Section 13.1.

### 16.10 Use `metadata` for ad-hoc fields, then filter on key presence

Every CRM resource carries a `metadata` JSON object that is yours to write to. Stash external-system identifiers, one-off custom fields, integration-specific flags, and anything the canonical schema does not model.

For people and deals, the API supports `?metadata__has_key=fieldName` — the right way to find every record carrying a specific custom key without enumerating values. Pair this with a naming convention (e.g. always write `external_id` rather than mixing `externalId` and `ext_id`) so the filter stays useful as the integration grows. See Section 13.3.

### 16.11 Check tag and invitation permissions separately

The RBAC model splits person actions from tag actions from invitation actions into separate buckets. A role with full `Ibl.CRM/Persons/*` rights does not automatically have `Ibl.CRM/Tags/*` or `Ibl.CRM/Invite/*` rights.

In practice this means:

- The "Add tag" affordance on a person card needs a tag-attach permission check, not a person-write check
- The "Invite to Platform" button needs an invite-action check
- The "Link existing user" button needs its own action check

Hide each control based on the specific permission it requires. Granting a generic "edit person" role is not enough to expose the full surface. See Section 14.

### 16.12 Handle the `active` flip

When the auto-link signal binds a person to a Platform user — either via explicit `link-user` or because the system matched on email at user creation — `Person.active` flips from `true` to `false` automatically.

Your UI may load the person in `active: true` state and see it return as `active: false` on the very next request, with no edit having taken place. This is intentional: an active person is one without a bound user, and binding consumes that state.

Code defensively:

- Do not cache `active` aggressively
- Treat the value as advisory for list filtering, not as an audit trail
- Re-fetch after any link or merge action before re-rendering the badge

See Section 8.4.

### 16.13 Use stable IDs to correlate across systems

When you import people from an external system, store the foreign identifier in `metadata.external_id` (or a convention of your choosing) at creation time. Do this in the same request that creates the record — not in a follow-up PATCH.

That foreign ID lets you re-match the record on subsequent syncs without relying on name or email, both of which can change without notice. The `?metadata__has_key=external_id` filter then becomes a cheap way to enumerate everything you originated, separately from records created in the CRM UI. See Section 16.10 and Section 12.

---
