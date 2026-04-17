# External Service Proxy API

## Purpose

The External Service Proxy API provides a unified interface for integrating third-party services like ElevenLabs and HeyGen into your application. This proxy layer handles authentication, request routing, and response formatting, allowing you to interact with multiple external services through a consistent API.

**Developer**

---

## Authentication

All requests require authentication via API key or token:

**API Key Authentication:**
```
Authorization: Api-Key <your-api-key>
```

**Token Authentication:**
```
Authorization: Token <your-token>
```

---

## Service Discovery

Before making API calls, use the discovery endpoints to find available services and their configurations.

### List All Services

Get all available external proxy services.

**Endpoint:**
```
GET /api/ai-proxy/orgs/{org}/services/
```

**Response:**
```json
[
  {
    "slug": "elevenlabs",
    "display_name": "ElevenLabs",
    "service_type": "tts",
    "is_enabled": true,
    "supports_async_jobs": false,
    "supports_streaming": true,
    "credential_name": "elevenlabs",
    "endpoint_count": 3
  },
  {
    "slug": "heygen",
    "display_name": "HeyGen",
    "service_type": "video",
    "is_enabled": true,
    "supports_async_jobs": true,
    "supports_streaming": false,
    "credential_name": "heygen",
    "endpoint_count": 6
  }
]
```

### Get Service Details

Get detailed information about a specific service including all available endpoints.

**Endpoint:**
```
GET /api/ai-proxy/orgs/{org}/services/{service}/
```

**Response:**
```json
{
  "slug": "elevenlabs",
  "display_name": "ElevenLabs",
  "base_url": "https://api.elevenlabs.io",
  "service_type": "tts",
  "auth_mode": "header",
  "is_enabled": true,
  "supports_async_jobs": false,
  "supports_streaming": true,
  "default_timeout_seconds": 60,
  "credential_name": "elevenlabs",
  "credential_policy": {
    "allow_tenant_key": true,
    "allow_platform_key": true,
    "default_source": "tenant",
    "fallback_to_platform_key": true
  },
  "credential_schema": {
    "key": "string"
  },
  "endpoints": [
    {
      "slug": "list-voices",
      "path_template": "/v1/voices",
      "http_method": "GET",
      "request_mode": "json",
      "response_mode": "json",
      "supports_streaming": false,
      "callback_mode": "none",
      "is_enabled": true
    },
    {
      "slug": "list-models",
      "path_template": "/v1/models",
      "http_method": "GET",
      "request_mode": "json",
      "response_mode": "json",
      "supports_streaming": false,
      "callback_mode": "none",
      "is_enabled": true
    },
    {
      "slug": "tts",
      "path_template": "/v1/text-to-speech/{voice_id}",
      "http_method": "POST",
      "request_mode": "json",
      "response_mode": "binary",
      "supports_streaming": true,
      "callback_mode": "none",
      "is_enabled": true
    }
  ]
}
```

**Frontend Example (JavaScript):**
```javascript
async function discoverServices() {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
      }
    }
  );
  return response.json();
}

async function getServiceDetails(serviceSlug) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/${serviceSlug}/`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
      }
    }
  );
  return response.json();
}

// Usage
const services = await discoverServices();
for (const service of services) {
  console.log(`Service: ${service.display_name}`);
  const details = await getServiceDetails(service.slug);
  console.log(`  Endpoints: ${details.endpoints.map(e => e.slug).join(', ')}`);
}
```

---

## Invoking Service Endpoints

Once you know the available services and endpoints, use the gateway endpoint to make API calls.

### Base URL

```
POST /api/ai-proxy/orgs/{org}/services/{service}/{action}/
```

**Parameters:**

| Parameter | Description |
|-----------|-------------|
| `org` | Your organization/tenant key |
| `service` | Service provider slug (from discovery) |
| `action` | Action/endpoint slug (from service details) |

### Request Format

All endpoints accept a JSON body with the following optional fields:

```json
{
  "body": {},           // JSON body to send to upstream API
  "query": {},          // Query parameters
  "headers": {},        // Additional headers
  "path_params": {},    // Path template parameters (for URLs like /v1/text-to-speech/{voice_id})
  "files": {}           // For multipart uploads
}
```

---

## Quick Reference

| Concept | Description |
|---------|-------------|
| **Service Discovery** | Use `/services/` endpoints to find available services and endpoints |
| **Gateway Endpoint** | `/api/ai-proxy/orgs/{org}/services/{service}/{action}/` |
| **Path Parameters** | Extract from `path_template` (e.g., `{voice_id}` in `/v1/text-to-speech/{voice_id}`) |
| **Response Modes** | `json`, `binary`, `passthrough`, `stream` |
| **Authentication** | API Key or Token in Authorization header |
| **Credentials** | Must be configured per service via admin panel |

---

## Related Pages

- [Integration Guide](integration.md) - Path templates, response modes, and dynamic client
- [ElevenLabs Integration](elevenlabs.md) - Text-to-speech endpoints
- [HeyGen Integration](heygen.md) - Video generation endpoints
- [Error Handling](errors.md) - Error responses and credential setup
