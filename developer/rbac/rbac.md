# RBAC Developer Guide

## Overview

IBL uses a role-based access control (RBAC) system that governs what users can do and see on a per-platform basis. Every permission check resolves to: **can this user perform this action on this resource?**

The system operates at two levels:
- **Action-level**: Can the user list/read/write/delete/action a resource? (gate for the operation)
- **Data-level**: Which fields can the user read or write on that resource? (field-level masking/validation)

All RBAC state is scoped to a **platform**. A user can have different roles on different platforms.

## Core Concepts

### Resource Paths

Resources are identified by hierarchical paths rooted at a platform:

```
/platforms/{platform_pk}/                              ← platform root
/platforms/{platform_pk}/mentors/                      ← mentor collection
/platforms/{platform_pk}/mentors/{mentor_id}/          ← specific mentor
/platforms/{platform_pk}/mentors/{id}/documents/       ← nested collection
/platforms/{platform_pk}/mentors/{id}/documents/{id}/  ← nested instance
/platforms/{platform_pk}/llms/openai/models/gpt-4o/    ← LLM model resource
/platforms/{platform_pk}/usergroups/                    ← teams collection
```

Matching is **hierarchical**: a policy's roles apply to the target resource and all its children. For example, a policy scoped to `/platforms/1/mentors/5/` also applies to `/platforms/1/mentors/5/documents/`, `/platforms/1/mentors/5/prompts/`, etc. A policy scoped to `/platforms/1/` applies to every resource under that platform.

### Actions

Actions follow the format `{Provider}.{Namespace}/{Resource}/{Operation}`:

```
Ibl.Mentor/Mentors/read       ← read a mentor
Ibl.Mentor/Mentors/list       ← list mentors
Ibl.Mentor/Mentors/write      ← update a mentor
Ibl.Mentor/Mentors/action     ← create a mentor
Ibl.Mentor/Mentors/delete     ← delete a mentor
Ibl.Mentor/Chat/action        ← chat with a mentor
Ibl.Mentor/Settings/read      ← read mentor settings
Ibl.Core/UserGroups/list      ← list teams
Ibl.Analytics/CanViewAnalytics/action ← view analytics
```

Wildcards are supported: `Ibl.Mentor/*` matches all mentor actions. `Ibl.*` matches everything.

### Data Actions

Data actions control **field-level** access. Format: `{Provider}.{Namespace}/{Resource}/{field_name}/{operation}`:

```
Ibl.Mentor/Settings/display_name/read    ← can read the display_name field
Ibl.Mentor/Settings/display_name/write   ← can write the display_name field
Ibl.Mentor/Settings/*/read               ← can read all settings fields
Ibl.Mentor/Mentors/*                     ← full field access on mentors
```

When a user lacks read permission on a field, the API returns an empty value (empty string, empty list, or empty dict) for that field. When a user lacks write permission, the API rejects the update with a 403 permission error..

### Roles

A **Role** (`RbacRole`) defines a set of `actions` and `data_actions`. Roles can be:
- **Global** (`platform=null`): Available across all platforms. Cannot be managed by platform admins.
- **Platform-scoped** (`platform` set): Belongs to a specific platform. Can be managed by that platform's admins.

Built-in roles include:

| Role | Purpose |
|------|---------|
| Tenant Admin | Full access |
| Students | Chat, read mentor settings, manage own artifacts |
| Mentor Viewer | Read-only access to mentor settings, documents, prompts |
| Mentor Editor | Full read/write to mentor settings, documents, prompts |
| Mentor Chat | Chat access (same actions as Students) |
| Student Mentor Creators | Can create mentors |
| Analytics Viewer | Can view analytics for teams they have the analytics permissions on|
| Notification Manager | Can send notifications to teams they have the notifications permission on, manage templates |
| Enrollment Manager | Course/program/pathway enrollments; can invite users to the platform |
| LLM Users | Can list available LLMs |
| LLM Model Access | Can use specific LLM models |
| List Users | Can list platform users |
| List Teams | Can list teams |
| Create Teams | Can create teams |
| Read Team | Can read team details |
| Edit Team | Can read/write team details |
| Billing Manager | Can manage credits |

### Policies

A **Policy** (`RbacPolicy`) binds a Role to Resources for specific Users/Groups:

```
Policy = Role + Resources + (Users and/or Groups)
```

- `role`: Which Role's actions apply
- `resources`: List of resource paths this policy applies to (e.g., `["/platforms/1/"]`)
- `users`: M2M to users directly assigned
- `groups`: M2M to RBAC groups (whose members inherit the policy)
- `name`: Unique per platform. Defaults to a UUID if not specified.

A user's effective permissions = union of all policies they belong to (directly or via groups).

### Groups

An **RBAC Group** (`RbacGroup`) is a collection of users. Assigning a group to a policy grants all group members that policy's permissions.

### Well-Known Roles

**Well-Known Roles** (`RbacWellKnownRole`) are supplemental policies applied dynamically based on context (not stored as policy assignments):

**Owner roles** — applied automatically when the requesting user owns the resource in question:

`mentor-owner`, `document-owner`, `prompt-owner`, `user-group-owner`, `artifact-owner`, `memory-owner`, `workflow-owner`, `mcp-server-owner`, `mcp-server-connection-owner`, `connected-service-owner`

For nested resources (documents, prompts), ownership of the parent mentor also grants the owner role.

Owner roles give the creator full control over their resources without requiring explicit policy assignments. For example, a student with the `Students` role who creates a mentor automatically gets `mentor-owner` permissions on that mentor so can fully manage it.

### Permission Evaluation Flow

```
1. User makes API request
2. System loads all user's policies (direct + group-based)
3. Well-known policies are added (e.g., "everyone")
4. For each policy:
   a. Does the policy's resource path match (hierarchically)?
   b. Does the policy's action list match the requested action?
5. If owner check is relevant, owner well-known role is loaded and checked
6. If any policy grants access → allowed
7. For data access: each field is checked individually against data_actions
```

## API Reference

All endpoints are under the `api/core/` prefix.

**Authentication:** All endpoints require authentication via a **Platform API Key** or **DM Token**. The authenticated identity must have the appropriate RBAC permissions for the operation.

### Roles CRUD

Manage role definitions. Action prefix: `Ibl.Core/Roles/`

| Method | Path | Required Action |
|--------|------|-----------------|
| GET | `/api/core/rbac/roles/?platform_key={key}` | `Ibl.Core/Roles/list` |
| POST | `/api/core/rbac/roles/` | `Ibl.Core/Roles/action` |
| GET | `/api/core/rbac/roles/{id}/?platform_key={key}` | `Ibl.Core/Roles/read` |
| PUT/PATCH | `/api/core/rbac/roles/{id}/` | `Ibl.Core/Roles/write` |
| DELETE | `/api/core/rbac/roles/{id}/?platform_key={key}` | `Ibl.Core/Roles/delete` |

**Query parameters (list/retrieve):**
| Param | Description |
|-------|-------------|
| `platform_key` | Required. Platform to scope roles to. |
| `include_global_roles` | Include global (platform=null) roles. Default: true. |
| `name` | Filter by role name (case-insensitive contains). |

**Request body (create/update):**
```json
{
  "name": "Custom Role",
  "platform_key": "my-platform",
  "actions": ["Ibl.Mentor/Mentors/read", "Ibl.Mentor/Chat/action"],
  "data_actions": ["Ibl.Mentor/Settings/*/read"]
}
```

**Response:**
```json
{
  "id": 42,
  "name": "Custom Role",
  "platform": 1,
  "actions": ["Ibl.Mentor/Mentors/read", "Ibl.Mentor/Chat/action"],
  "data_actions": ["Ibl.Mentor/Settings/*/read"]
}
```

---

### Policies CRUD

Manage policy assignments. Action prefix: `Ibl.Core/Policies/`

| Method | Path | Required Action |
|--------|------|-----------------|
| GET | `/api/core/rbac/policies/?platform_key={key}` | `Ibl.Core/Policies/list` |
| POST | `/api/core/rbac/policies/` | `Ibl.Core/Policies/action` |
| GET | `/api/core/rbac/policies/{id}/?platform_key={key}` | `Ibl.Core/Policies/read` |
| PUT/PATCH | `/api/core/rbac/policies/{id}/` | `Ibl.Core/Policies/write` |
| DELETE | `/api/core/rbac/policies/{id}/?platform_key={key}` | `Ibl.Core/Policies/delete` |

**Query parameters (list):**
| Param | Description |
|-------|-------------|
| `platform_key` | Required. |
| `role_id` | Filter by role. |
| `name` | Filter by policy name (case-insensitive contains). |
| `username` | Filter by user (searches direct assignments and group memberships). |
| `email` | Filter by email (same search as username). |
| `group` | Filter by group name (case-insensitive exact). |
| `include_users` | Include user details in response. Default: true. |
| `include_groups` | Include group details in response. Default: true. |

**Request body (create/update):**
```json
{
  "platform_key": "my-platform",
  "name": "My Policy",
  "role_id": 42,
  "resources": ["/platforms/1/mentors/"],
  "user_ids": [101, 102],
  "group_ids": [5]
}
```

Resources must start with `/platforms/` and are normalized to end with `/`.


---

### Groups CRUD

Manage user groups for policy assignment. Action prefix: `Ibl.Core/Groups/`

| Method | Path | Required Action |
|--------|------|-----------------|
| GET | `/api/core/rbac/groups/?platform_key={key}` | `Ibl.Core/Groups/list` |
| POST | `/api/core/rbac/groups/` | `Ibl.Core/Groups/action` |
| GET | `/api/core/rbac/groups/{id}/?platform_key={key}` | `Ibl.Core/Groups/read` |
| PUT/PATCH | `/api/core/rbac/groups/{id}/` | `Ibl.Core/Groups/write` |
| DELETE | `/api/core/rbac/groups/{id}/?platform_key={key}` | `Ibl.Core/Groups/delete` |

**Query parameters (list):**
| Param | Description |
|-------|-------------|
| `platform_key` | Required. |
| `owner` | Filter by owner username. |
| `name` | Filter by group name (case-insensitive contains). |
| `username` | Filter by member username. |
| `email` | Filter by member email. |
| `include_users` | Include user details. Default: true. |

**Request body (create/update):**
```json
{
  "platform_key": "my-platform",
  "name": "Editors Group",
  "unique_id": "editors-group",
  "description": "Users with editor access",
  "user_ids": [101, 102]
}
```


---

### Permission Check

Check what permissions the authenticated user has on a set of resources. No specific RBAC action required — any authenticated user can check their own permissions.

```
POST /api/core/rbac/permissions/check/
```

**Request body:**
```json
{
  "platform_key": "my-platform",
  "resources": [
    "/mentors/",
    "/mentors/42/",
    "/mentors/42/documents/",
    "/usergroups/"
  ]
}
```

**Response:**
```json
{
  "/mentors/": {
    "list": true,
    "create": false,
    "chat": true
  },
  "/mentors/42/": {
    "read": true,
    "write": true,
    "delete": true,
    "chat": true,
    "show_settings": true,
    "share_mentor": false,
    "view_prompts": true,
    "view_chat_history": true
  },
  "/mentors/42/documents/": {
    "list": true,
    "create": true
  },
  "/usergroups/": {
    "list": true,
    "create": false
  }
}
```

The permissions returned depend on the resource type. See the [Supported Resource Types](#supported-resource-types) section for what each resource type returns.

This endpoint accounts for ownership: if the user owns a mentor, the owner well-known role is applied automatically.

---

### Mentor Access Management

Grant/revoke user access to specific mentors.

| Method | Path | Required Action |
|--------|------|-----------------|
| POST | `/api/core/rbac/mentor-access/` | `Ibl.Mentor/ShareMentor/action` |
| GET | `/api/core/rbac/mentor-access/?platform_key={key}&mentor_id={id}` | `Ibl.Mentor/ShareMentor/read` |

Checked against the mentor's resource path (`/platforms/{pk}/mentors/{mentor_id}/`). Owner role applies if the user created the mentor.

**POST body:**
```json
{
  "platform_key": "my-platform",
  "mentor_id": "unique-mentor-uuid",
  "role": "viewer",
  "users_to_add": [101, 102],
  "users_to_remove": [103],
  "groups_to_add": [5],
  "groups_to_remove": [6]
}

```
You can optionally use `usernames_to_add` and `emails_to_add` instead of `users_to_add` if you don't have id's.

Valid `role` values: `"chat"`, `"viewer"`, `"editor"`

**GET response:** Returns all access policies for the mentor.

---

### Team (UserGroup) Sharing

Share teams with users/groups at different access levels.

| Method | Path | Required Action |
|--------|------|-----------------|
| POST | `/api/core/rbac/teams/access/` | `Ibl.Core/ShareUserGroups/action` |
| GET | `/api/core/rbac/teams/access/?platform_key={key}&usergroup_id={id}` | `Ibl.Core/ShareUserGroups/read` |

Checked against the team's resource path (`/platforms/{pk}/usergroups/{id}/`). Owner role applies if the user owns the team.

**POST body:**
```json
{
  "platform_key": "my-platform",
  "usergroup_id": 10,
  "role": "edit",
  "users_to_add": [101],
  "users_to_remove": [],
  "groups_to_add": [],
  "groups_to_remove": []
}
```

Valid `role` values: `"read"`, `"edit"`, `"view analytics"`, `"send notifications"`

Secondary policies (e.g., List Teams, List Users) are automatically managed.

---

### User Policy Bulk Update

Bulk add/remove/replace named policies for users.

| Method | Path | Required Action |
|--------|------|-----------------|
| PUT | `/api/core/platform/users/policies/` | `Ibl.Core/UserPolicies/write` |

**Request body (list of updates):**
```json
[
  {
    "user_id": 101,
    "platform_key": "my-platform",
    "policies_to_add": ["Analytics Viewer", "Mentor Chat"],
    "policies_to_remove": ["Notification Manager"]
  }
]
```

- `policies_to_add`: Add user to these named policies
- `policies_to_remove`: Remove user from these named policies
- `policies_to_set`: Replace all policy assignments with this list (removals happen before additions)

Only policies from the platform's assignable list are allowed.

---

### Student Mentor Creation Toggle

Control whether students can create mentors on a platform. Requires platform admin access (no additional RBAC action).

```
POST /api/core/rbac/student-mentor-creation/set/
GET  /api/core/rbac/student-mentor-creation/status/?platform_key={key}
```

**POST body:**
```json
{
  "platform_key": "my-platform",
  "allow_students_to_create_mentors": true
}
```

This adds/removes the Students group from the Mentor Creator policy.

---

### Student LLM Access

Control which LLM models students can use. Requires platform admin access (no additional RBAC action).

```
POST /api/core/rbac/student-llm-access/set/
GET  /api/core/rbac/student-llm-access/status/?platform_key={key}
```

**POST body:**
```json
{
  "platform_key": "my-platform",
  "llm_resources": [
    "llms/openai/models/gpt-4o",
    "llms/anthropic/models/claude-3-5-sonnet"
  ]
}
```

Resources are relative paths that get prefixed with `/platforms/{pk}/`.

---

## Response Permission Metadata

API responses typically include a `permissions` object:

```json
{
  "id": 42,
  "display_name": "My Mentor",
  "description": "",
  "permissions": {
    "field": {
      "display_name": {"read": true, "write": true},
      "description": {"read": true, "write": false}
    },
    "object": {
      "delete": false,
      "write": true
    }
  }
}
```

- `permissions.field`: Per-field read/write permissions
- `permissions.object`: Object-level delete/write permissions

Fields the user cannot read are masked (returned as `""`, `[]`, or `{}` depending on type) and the `read` permission will be `False`. In the example above, `description` is empty because the user lacks read access.

This metadata enables UIs to:
- Hide fields the user can't see
- Disable editing for read-only fields
- Show/hide delete buttons based on object permissions

---

## Supported Resource Types

### Collection-level operations
Resources ending in a type name (e.g., `/platforms/1/mentors/`):

| Resource Type | Operations |
|---------------|-----------|
| `mentors` | list, create, chat, web_search, attach_document, voice_record, voice_call, export_chat_history, view_chat_history, view_analytics, view_prompts, share, sell_mentor |
| `prompts` | list, create |
| `documents` | list, create |
| `tools` | list, create |
| `settings` | list, create |
| `llms` | list, create |
| `mcpservers` | list, create |
| `usergroups` | list, create |
| `users` | list, write |
| `groups` | list, create |
| `policies` | list, create |
| `roles` | list, create |

### Instance-level operations
Resources ending in an ID (e.g., `/platforms/1/mentors/42/`):

| Resource Type | Operations |
|---------------|-----------|
| `mentors` | read, write, delete, chat, web_search, attach_document, voice_record, voice_call, export_chat_history, view_chat_history, view_analytics, view_prompts, show_settings, share_mentor, read_shared_mentor, sell_mentor, can_use_embed, view_moderation_logs, view_safety_logs, view_disclaimers, view_prompts_menu, view_tools_menu, view_disclaimers_menu |
| `prompts` | read, write, delete |
| `documents` | read, write, delete |
| `tools` | read, write, delete |
| `settings` | read, write, delete |
| `llms` | read, write, delete |
| `mcpservers` | read, write, delete |
| `usergroups` | read, write, delete, share_usergroup, read_shared_usergroup |
| `platforms` | can_send_notifications, can_view_analytics, can_manage_users, can_invite |

---

## How Policies Combine

Permissions are **additive**. If any policy grants an action, it's allowed.

Example: A user in the "Students" group with an additional "Mentor Editor" policy on `/platforms/1/mentors/5/` gets:
- Student-level permissions on all platform resources (chat, list mentors, etc.)
- Editor-level permissions on mentor 5 specifically (write settings, manage documents, etc.)

## Action Pattern Matching

Actions support `*` as a wildcard to match any segment:

| Pattern | Matches |
|---------|---------|
| `Ibl.Mentor/Settings/read` | Exactly `Ibl.Mentor/Settings/read` |
| `Ibl.Mentor/Settings/*` | `Ibl.Mentor/Settings/read`, `Ibl.Mentor/Settings/write`, etc. |
| `Ibl.Mentor/*` | Any mentor action |
| `Ibl.*` | Everything (Tenant Admin) |

Data action wildcards work the same way:

| Pattern | Matches |
|---------|---------|
| `Ibl.Mentor/Settings/display_name/read` | Read the display_name field |
| `Ibl.Mentor/Settings/*/read` | Read any settings field |
| `Ibl.Mentor/Settings/*` | Read and write any settings field |
