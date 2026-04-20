# User Platform Metadata API

## Overview

The User Platform Metadata API allows you to store and manage per-user, per-platform metadata such as user preferences, application settings, feature flags, and custom key-value data. Each user can have different metadata for different platforms, enabling flexible, context-aware applications.

**Developer**

---

## Purpose

Store and manage per-user, per-platform metadata including:
- **User Preferences** - Theme settings, language preferences, notification settings
- **Application Settings** - UI state, feature toggles, onboarding progress
- **Feature Flags** - Progressive rollout, A/B testing, beta feature access
- **Custom Data** - Any key-value pairs your application needs to persist per user

---

## Authentication

All requests require token-based authentication:

```
Authorization: Token <manager_token>
```

Include this header in every request to the API.

---

## Base URL

```
/api/core/users/platform-metadata/
```

### Query Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `platform_key` | Yes | Platform identifier (e.g., `acme-corp`) |
| `username` | No | Target user (admin only, defaults to authenticated user) |

---

## GET - Retrieve Metadata

Retrieve all metadata for a user on a specific platform.

**Endpoint:**
```
GET /api/core/users/platform-metadata/?platform_key={platform_key}
```

**Request Example:**
```javascript
const response = await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
  {
    headers: {
      'Authorization': `Token ${token}`
    }
  }
);

const data = await response.json();
console.log(data);
```

**Response:**
```json
{
  "username": "john_doe",
  "platform_key": "acme-corp",
  "metadata": {
    "theme": "dark",
    "language": "en",
    "notifications_enabled": true,
    "onboarding_completed": true
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-20T14:45:00Z"
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `username` | string | Username of the user |
| `platform_key` | string | Platform identifier |
| `metadata` | object | Key-value pairs of user metadata |
| `created_at` | string | ISO 8601 timestamp of creation |
| `updated_at` | string | ISO 8601 timestamp of last update |

---

## PATCH - Update or Delete Specific Keys

Update specific metadata keys or delete keys without affecting other metadata. You can use `metadata` to add/update keys, `delete_keys` to remove keys, or both in the same request.

**Endpoint:**
```
PATCH /api/core/users/platform-metadata/?platform_key={platform_key}
```

### Update Keys

Add new keys or update existing ones:

```javascript
await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
  {
    method: 'PATCH',
    headers: {
      'Authorization': `Token ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      metadata: {
        theme: 'light',
        fontSize: '16px',
        newSetting: 'value'
      }
    })
  }
);
```

**Result:** The specified keys are added or updated. All other existing metadata remains unchanged.

### Delete Keys

Remove specific keys from metadata:

```javascript
await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
  {
    method: 'PATCH',
    headers: {
      'Authorization': `Token ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      delete_keys: ['oldSetting', 'deprecatedFlag']
    })
  }
);
```

**Result:** The specified keys are removed. All other existing metadata remains unchanged.

### Update and Delete Together

Perform both operations in a single request:

```javascript
await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
  {
    method: 'PATCH',
    headers: {
      'Authorization': `Token ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      metadata: { theme: 'dark', newFeature: true },
      delete_keys: ['legacyTheme', 'oldFeature']
    })
  }
);
```

**Result:** Keys in `delete_keys` are removed first, then keys in `metadata` are added/updated.

**Response:**
Returns the updated metadata object with the same structure as GET.

---

## PUT - Replace All Metadata

Completely replace all existing metadata with new data. This operation removes all existing keys and replaces them with the provided metadata.

**Endpoint:**
```
PUT /api/core/users/platform-metadata/?platform_key={platform_key}
```

**Request Example:**
```javascript
await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
  {
    method: 'PUT',
    headers: {
      'Authorization': `Token ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      metadata: {
        theme: 'system',
        language: 'es'
      }
    })
  }
);
```

**Result:** All previous metadata is removed and replaced with the new metadata object containing only `theme` and `language`.

**Response:**
Returns the updated metadata object with the same structure as GET.

**Warning:** This operation is destructive. All existing metadata keys not included in the request will be permanently deleted.

---

## DELETE - Clear All Metadata

Remove all metadata for a user on a specific platform.

**Endpoint:**
```
DELETE /api/core/users/platform-metadata/?platform_key={platform_key}
```

**Request Example:**
```javascript
await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
  {
    method: 'DELETE',
    headers: {
      'Authorization': `Token ${token}`
    }
  }
);
```

**Response:**
```
204 No Content
```

**Result:** All metadata for the user on the specified platform is permanently deleted.

---

## Admin Operations

Tenant administrators can access and modify metadata for any user on their platform by adding the `username` query parameter.

### Get Another User's Metadata

```javascript
const response = await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}&username=other_user`,
  {
    headers: {
      'Authorization': `Token ${adminToken}`
    }
  }
);

const data = await response.json();
```

### Update Another User's Metadata

```javascript
await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}&username=other_user`,
  {
    method: 'PATCH',
    headers: {
      'Authorization': `Token ${adminToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      metadata: {
        tier: 'premium',
        features: ['advanced-analytics', 'priority-support']
      }
    })
  }
);
```

### Replace Another User's Metadata

```javascript
await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}&username=other_user`,
  {
    method: 'PUT',
    headers: {
      'Authorization': `Token ${adminToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      metadata: {
        accountType: 'enterprise',
        region: 'US'
      }
    })
  }
);
```

### Delete Another User's Metadata

```javascript
await fetch(
  `/api/core/users/platform-metadata/?platform_key=${platformKey}&username=other_user`,
  {
    method: 'DELETE',
    headers: {
      'Authorization': `Token ${adminToken}`
    }
  }
);
```

**Permission Required:** Tenant admin role on the specified platform.

---

## Use Cases & Examples

### Example 1: User Preferences

Store and retrieve user interface preferences:

```javascript
// Save user's theme preference
await fetch(`/api/core/users/platform-metadata/?platform_key=my-app`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Token ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    metadata: {
      theme: 'dark',
      fontSize: '14px',
      sidebarCollapsed: false,
      language: 'en'
    }
  })
});

// Retrieve preferences on next login
const response = await fetch(
  `/api/core/users/platform-metadata/?platform_key=my-app`,
  { headers: { 'Authorization': `Token ${token}` } }
);

const { metadata } = await response.json();
applyTheme(metadata.theme);
setFontSize(metadata.fontSize);
```

### Example 2: Feature Flags

Manage user-specific feature access:

```javascript
// Admin enables beta feature for a user
await fetch(
  `/api/core/users/platform-metadata/?platform_key=my-app&username=beta_tester`,
  {
    method: 'PATCH',
    headers: {
      'Authorization': `Token ${adminToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      metadata: {
        features: {
          newDashboard: true,
          aiAssistant: true,
          advancedReports: false
        }
      }
    })
  }
);

// User checks if feature is enabled
const response = await fetch(
  `/api/core/users/platform-metadata/?platform_key=my-app`,
  { headers: { 'Authorization': `Token ${token}` } }
);

const { metadata } = await response.json();
if (metadata.features?.newDashboard) {
  showNewDashboard();
}
```

### Example 3: Onboarding Progress

Track user onboarding state:

```javascript
// Update onboarding progress as user completes steps
await fetch(`/api/core/users/platform-metadata/?platform_key=my-app`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Token ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    metadata: {
      onboarding: {
        welcomeComplete: true,
        profileComplete: true,
        firstTaskComplete: false,
        tourComplete: false
      },
      lastStep: 'profile',
      lastStepTimestamp: new Date().toISOString()
    }
  })
});

// On next login, resume from last step
const response = await fetch(
  `/api/core/users/platform-metadata/?platform_key=my-app`,
  { headers: { 'Authorization': `Token ${token}` } }
);

const { metadata } = await response.json();
if (!metadata.onboarding?.firstTaskComplete) {
  showOnboardingStep('first-task');
}
```

### Example 4: Multi-Platform Settings

Different settings for different platforms:

```javascript
// User has different preferences on different platforms
// Platform A - Corporate
await fetch(`/api/core/users/platform-metadata/?platform_key=corporate`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Token ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    metadata: {
      theme: 'light',
      notifications: 'all',
      language: 'en'
    }
  })
});

// Platform B - Personal
await fetch(`/api/core/users/platform-metadata/?platform_key=personal`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Token ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    metadata: {
      theme: 'dark',
      notifications: 'important-only',
      language: 'es'
    }
  })
});
```

---

## Error Responses

| Status | Error Message | Cause | Solution |
|--------|---------------|-------|----------|
| 400 | `{"error": "platform_key query parameter is required"}` | Missing `platform_key` in query | Add `?platform_key=your-platform` to URL |
| 400 | `{"non_field_errors": ["Provide 'metadata' to update or 'delete_keys' to remove keys."]}` | PATCH request with neither `metadata` nor `delete_keys` | Include at least one of `metadata` or `delete_keys` in request body |
| 401 | `{"detail": "Authentication credentials were not provided."}` | Missing or invalid authorization header | Add `Authorization: Token <token>` header |
| 403 | `{"error": "Only tenant admins can access other users' metadata"}` | Non-admin trying to use `username` parameter | Remove `username` parameter or use admin token |
| 404 | `{"error": "Platform not found"}` | Invalid `platform_key` | Verify the platform key exists |
| 404 | `{"error": "User not found"}` | Invalid `username` (admin operation) | Verify the username exists on the platform |

**Common Issues:**

1. **Missing platform_key:**
   ```javascript
   // ❌ Wrong
   fetch('/api/core/users/platform-metadata/')

   // ✅ Correct
   fetch('/api/core/users/platform-metadata/?platform_key=my-platform')
   ```

2. **Empty PATCH request:**
   ```javascript
   // ❌ Wrong
   body: JSON.stringify({})

   // ✅ Correct
   body: JSON.stringify({ metadata: { key: 'value' } })
   // or
   body: JSON.stringify({ delete_keys: ['key'] })
   ```

3. **Unauthorized admin access:**
   ```javascript
   // ❌ Wrong - regular user trying to access another user
   fetch('/api/core/users/platform-metadata/?platform_key=my-platform&username=other_user', {
     headers: { 'Authorization': `Token ${regularUserToken}` }
   })

   // ✅ Correct - use admin token
   fetch('/api/core/users/platform-metadata/?platform_key=my-platform&username=other_user', {
     headers: { 'Authorization': `Token ${adminToken}` }
   })
   ```

---

## Best Practices

### 1. PATCH vs PUT

**Use PATCH** when:
- Updating specific settings without affecting others
- Adding new keys to existing metadata
- Removing specific keys with `delete_keys`
- Building up metadata incrementally

**Use PUT** when:
- Resetting all settings to defaults
- Migrating metadata structure (replacing old schema with new)
- Ensuring no legacy keys remain

### 2. Metadata Structure

**Keep metadata flat when possible:**
```javascript
// ✅ Good - flat structure
{
  "theme": "dark",
  "language": "en",
  "notifications": true
}

// ⚠️ Acceptable - nested for logical grouping
{
  "ui": {
    "theme": "dark",
    "fontSize": "14px"
  },
  "notifications": {
    "email": true,
    "push": false
  }
}

// ❌ Avoid - deeply nested
{
  "settings": {
    "user": {
      "preferences": {
        "ui": {
          "theme": "dark"
        }
      }
    }
  }
}
```

**Use consistent key naming:**
```javascript
// ✅ Good - consistent naming
{
  "theme_preference": "dark",
  "language_preference": "en",
  "notification_preference": "all"
}

// ❌ Avoid - inconsistent naming
{
  "themePreference": "dark",
  "lang": "en",
  "notifications-setting": "all"
}
```

### 3. Performance Considerations

**Cache frequently accessed metadata:**
```javascript
// Cache metadata in application state
let cachedMetadata = null;

async function getMetadata(platformKey, token) {
  if (cachedMetadata) {
    return cachedMetadata;
  }

  const response = await fetch(
    `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
    { headers: { 'Authorization': `Token ${token}` } }
  );

  const data = await response.json();
  cachedMetadata = data.metadata;
  return cachedMetadata;
}

// Invalidate cache after updates
async function updateMetadata(platformKey, token, updates) {
  await fetch(`/api/core/users/platform-metadata/?platform_key=${platformKey}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Token ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ metadata: updates })
  });

  cachedMetadata = null; // Invalidate cache
}
```

**Batch updates:**
```javascript
// ✅ Good - single request
await fetch(`/api/core/users/platform-metadata/?platform_key=${platformKey}`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Token ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    metadata: {
      theme: 'dark',
      language: 'en',
      notifications: true
    }
  })
});

// ❌ Avoid - multiple requests
await updateSetting('theme', 'dark');
await updateSetting('language', 'en');
await updateSetting('notifications', true);
```

### 4. Security Considerations

**Never store sensitive data:**
```javascript
// ❌ NEVER store passwords, tokens, or secrets
{
  "password": "user_password",
  "apiKey": "secret_key",
  "creditCard": "1234-5678-9012-3456"
}

// ✅ Store non-sensitive preferences only
{
  "theme": "dark",
  "language": "en",
  "hasCompletedOnboarding": true
}
```

**Validate metadata on the frontend:**
```javascript
function validateMetadata(metadata) {
  const allowedKeys = ['theme', 'language', 'notifications', 'fontSize'];

  for (const key of Object.keys(metadata)) {
    if (!allowedKeys.includes(key)) {
      throw new Error(`Invalid metadata key: ${key}`);
    }
  }

  return metadata;
}

const updates = validateMetadata({ theme: 'dark', language: 'en' });
await updateMetadata(platformKey, token, updates);
```

### 5. Error Handling

**Always handle errors gracefully:**
```javascript
async function safeGetMetadata(platformKey, token) {
  try {
    const response = await fetch(
      `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
      { headers: { 'Authorization': `Token ${token}` } }
    );

    if (!response.ok) {
      console.error('Failed to fetch metadata:', response.status);
      return getDefaultMetadata(); // Fallback to defaults
    }

    const data = await response.json();
    return data.metadata;
  } catch (error) {
    console.error('Error fetching metadata:', error);
    return getDefaultMetadata(); // Fallback to defaults
  }
}

function getDefaultMetadata() {
  return {
    theme: 'light',
    language: 'en',
    notifications: true
  };
}
```

### 6. Migration Strategy

**Migrating metadata schema:**
```javascript
async function migrateMetadata(platformKey, token) {
  // Get current metadata
  const response = await fetch(
    `/api/core/users/platform-metadata/?platform_key=${platformKey}`,
    { headers: { 'Authorization': `Token ${token}` } }
  );

  const { metadata } = await response.json();

  // Transform to new schema
  const newMetadata = {
    ui: {
      theme: metadata.theme || 'light',
      fontSize: metadata.fontSize || '14px'
    },
    preferences: {
      language: metadata.language || 'en',
      notifications: metadata.notifications || true
    }
  };

  // Replace with new schema
  await fetch(`/api/core/users/platform-metadata/?platform_key=${platformKey}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Token ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ metadata: newMetadata })
  });
}
```

---

## Related Pages

- [Chat Metadata](../agents/chat-metadata.md) - Pass context alongside chat messages
- [RBAC](../rbac/rbac.md) - Role-based access control for permissions
- [Notifications API](../applications/notifications.md) - Send notifications to users
