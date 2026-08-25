# HeyGen Integration

## Overview

HeyGen provides AI-powered video generation capabilities with avatars and voices. The External Service Proxy API allows you to create videos programmatically using HeyGen's templates and avatars.

**Developer**

---

## Prerequisites

- HeyGen API credentials configured for your organization
- See [Credential Setup](errors.md#setting-up-credentials) for configuration details
- Valid API key or token for authentication

---

## List Templates

Get all available video templates.

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/heygen/list-templates/
```

**Request:**
```json
{}
```

**Response:**
```json
{
  "data": {
    "templates": [
      {
        "template_id": "abc123",
        "name": "Training Template"
      }
    ]
  }
}
```

**Frontend Example:**
```javascript
async function listTemplates(org, apiKey) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/heygen/list-templates/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({})
    }
  );

  const data = await response.json();
  return data.data.templates;
}
```

---

## List Avatars

Get all available avatars.

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/heygen/list-avatars/
```

**Request:**
```json
{}
```

**Response:**
```json
{
  "data": {
    "avatars": [
      {
        "avatar_id": "Abigail_expressive_2024112501",
        "avatar_name": "Abigail (Upper Body)"
      }
    ]
  }
}
```

**Frontend Example:**
```javascript
async function listAvatars(org, apiKey) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/heygen/list-avatars/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({})
    }
  );

  const data = await response.json();
  return data.data.avatars;
}
```

---

## List Voices

Get all available voices.

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/heygen/list-voices/
```

**Request:**
```json
{}
```

**Response:**
```json
{
  "data": {
    "voices": [
      {
        "voice_id": "f38a635bee7a4d1f9b0a654a31d050d2",
        "name": "Chill Brian",
        "language": "English"
      }
    ]
  }
}
```

**Frontend Example:**
```javascript
async function listVoices(org, apiKey) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/heygen/list-voices/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({})
    }
  );

  const data = await response.json();
  return data.data.voices;
}
```

---

## Generate Video

### Avatar-based Video Generation

Generate a video using an avatar (no template required).

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/heygen/generate-video/
```

**Request:**
```json
{
  "body": {
    "test": true,
    "video_inputs": [
      {
        "character": {
          "type": "avatar",
          "avatar_id": "Abigail_expressive_2024112501",
          "avatar_style": "normal"
        },
        "voice": {
          "type": "text",
          "input_text": "Hello, this is a test video.",
          "voice_id": "f38a635bee7a4d1f9b0a654a31d050d2"
        }
      }
    ],
    "dimension": {
      "width": 1280,
      "height": 720
    }
  }
}
```

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `test` | boolean | No | Set to true for test mode (no credits charged) |
| `video_inputs` | array | Yes | Array of video input configurations |
| `video_inputs[].character` | object | Yes | Avatar configuration |
| `video_inputs[].character.type` | string | Yes | Must be "avatar" |
| `video_inputs[].character.avatar_id` | string | Yes | Avatar ID from List Avatars |
| `video_inputs[].character.avatar_style` | string | Yes | Avatar style (e.g., "normal") |
| `video_inputs[].voice` | object | Yes | Voice configuration |
| `video_inputs[].voice.type` | string | Yes | Must be "text" |
| `video_inputs[].voice.input_text` | string | Yes | The script text |
| `video_inputs[].voice.voice_id` | string | Yes | Voice ID from List Voices |
| `dimension` | object | Yes | Video dimensions |
| `dimension.width` | number | Yes | Video width in pixels |
| `dimension.height` | number | Yes | Video height in pixels |

**Response:**
```json
{
  "data": {
    "video_id": "video_123abc"
  }
}
```

**Frontend Example:**
```javascript
async function generateVideo(org, apiKey, avatarId, voiceId, script) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/heygen/generate-video/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        body: {
          test: true,
          video_inputs: [{
            character: {
              type: 'avatar',
              avatar_id: avatarId,
              avatar_style: 'normal'
            },
            voice: {
              type: 'text',
              input_text: script,
              voice_id: voiceId
            }
          }],
          dimension: { width: 1280, height: 720 }
        }
      })
    }
  );

  const result = await response.json();
  return result.data?.video_id;
}
```

---

### Template-based Video Generation

Generate a video from a template.

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/heygen/generate-template-video/
```

**Path Template:** `/v2/template/{template_id}/generate`

**Required Path Parameters:**
- `template_id` - The ID of the template to use (get from List Templates endpoint)

**Request:**
```json
{
  "path_params": {
    "template_id": "your-template-id"
  },
  "body": {
    "test": true,
    "caption": false,
    "variables": {
      "script": {
        "name": "script",
        "type": "text",
        "properties": {
          "content": "Hello, this is a test video."
        }
      }
    }
  }
}
```

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `test` | boolean | No | Set to true for test mode (no credits charged) |
| `caption` | boolean | No | Enable or disable captions |
| `variables` | object | Yes | Template variables (depends on template) |

**Response:**
```json
{
  "data": {
    "video_id": "video_123abc"
  }
}
```

**Frontend Example:**
```javascript
async function generateTemplateVideo(org, apiKey, templateId, script) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/heygen/generate-template-video/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        path_params: { template_id: templateId },
        body: {
          test: true,
          caption: false,
          variables: {
            script: {
              name: 'script',
              type: 'text',
              properties: {
                content: script
              }
            }
          }
        }
      })
    }
  );

  const result = await response.json();
  return result.data?.video_id;
}
```

---

## Video Status

Check the status of a video generation job.

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/heygen/video-status/
```

**Request:**
```json
{
  "query": {
    "video_id": "video_123abc"
  }
}
```

**Response (Processing):**
```json
{
  "data": {
    "status": "processing"
  }
}
```

**Response (Completed):**
```json
{
  "data": {
    "status": "completed",
    "video_url": "https://..."
  }
}
```

**Response (Failed):**
```json
{
  "data": {
    "status": "failed",
    "error": "Error message"
  }
}
```

**Status Polling Example:**
```javascript
async function pollVideoStatus(org, apiKey, videoId, maxAttempts = 60, interval = 5000) {
  for (let i = 0; i < maxAttempts; i++) {
    const response = await fetch(
      `/api/ai-proxy/orgs/${org}/services/heygen/video-status/`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Api-Key ${apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          query: { video_id: videoId }
        })
      }
    );

    const result = await response.json();
    const status = result.data?.status;

    if (status === 'completed') {
      return result.data.video_url;
    } else if (status === 'failed') {
      throw new Error(result.data.error || 'Video generation failed');
    }

    // Wait before next check
    await new Promise(resolve => setTimeout(resolve, interval));
  }

  throw new Error('Video generation timed out');
}
```

---

## Complete Workflow Example

This example demonstrates a complete workflow: discovering avatars and voices, generating a video, and polling for completion.

```javascript
async function heygenWorkflow(org, apiKey, script) {
  try {
    // Step 1: Get available avatars
    const avatarsResponse = await fetch(
      `/api/ai-proxy/orgs/${org}/services/heygen/list-avatars/`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Api-Key ${apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({})
      }
    );
    const avatarsData = await avatarsResponse.json();
    const avatars = avatarsData.data.avatars;

    console.log('Available avatars:', avatars.map(a => a.avatar_name));

    // Step 2: Get available voices
    const voicesResponse = await fetch(
      `/api/ai-proxy/orgs/${org}/services/heygen/list-voices/`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Api-Key ${apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({})
      }
    );
    const voicesData = await voicesResponse.json();
    const voices = voicesData.data.voices;

    console.log('Available voices:', voices.map(v => v.name));

    // Step 3: Select avatar and voice
    const selectedAvatar = avatars[0];
    const selectedVoice = voices[0];

    // Step 4: Start video generation
    const generateResponse = await fetch(
      `/api/ai-proxy/orgs/${org}/services/heygen/generate-video/`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Api-Key ${apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          body: {
            test: true,
            video_inputs: [{
              character: {
                type: 'avatar',
                avatar_id: selectedAvatar.avatar_id,
                avatar_style: 'normal'
              },
              voice: {
                type: 'text',
                input_text: script,
                voice_id: selectedVoice.voice_id
              }
            }],
            dimension: { width: 1280, height: 720 }
          }
        })
      }
    );

    const generateResult = await generateResponse.json();
    const videoId = generateResult.data?.video_id;

    if (!videoId) {
      throw new Error('Failed to start video generation');
    }

    console.log('Video generation started. Video ID:', videoId);

    // Step 5: Poll for completion
    const videoUrl = await pollVideoStatus(org, apiKey, videoId);

    console.log('Video completed! URL:', videoUrl);
    return videoUrl;

  } catch (error) {
    console.error('Workflow failed:', error);
    throw error;
  }
}

// Helper function for polling
async function pollVideoStatus(org, apiKey, videoId, maxAttempts = 60, interval = 5000) {
  for (let i = 0; i < maxAttempts; i++) {
    const response = await fetch(
      `/api/ai-proxy/orgs/${org}/services/heygen/video-status/`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Api-Key ${apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          query: { video_id: videoId }
        })
      }
    );

    const result = await response.json();
    const status = result.data?.status;

    console.log(`Check ${i + 1}/${maxAttempts}: ${status}`);

    if (status === 'completed') {
      return result.data.video_url;
    } else if (status === 'failed') {
      throw new Error(result.data.error || 'Video generation failed');
    }

    await new Promise(resolve => setTimeout(resolve, interval));
  }

  throw new Error('Video generation timed out');
}

// Usage
heygenWorkflow('my-org', 'my-api-key', 'Hello! This is a demonstration of HeyGen video generation.')
  .then(videoUrl => console.log('Final video URL:', videoUrl))
  .catch(error => console.error('Failed:', error));
```

---

## Related Pages

- [Overview](overview.md) - Authentication and service discovery
- [Integration Guide](integration.md) - Path templates and dynamic client
- [ElevenLabs Integration](elevenlabs.md) - Text-to-speech endpoints
- [Error Handling](errors.md) - Error responses and credential setup
