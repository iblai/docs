# ElevenLabs Integration

## Overview

ElevenLabs provides advanced text-to-speech (TTS) capabilities with natural-sounding voices. The External Service Proxy API allows you to integrate ElevenLabs TTS functionality into your application.

**Developer**

---

## Prerequisites

- ElevenLabs API credentials configured for your organization
- See [Credential Setup](errors.md#setting-up-credentials) for configuration details
- Valid API key or token for authentication

---

## List Voices

Get all available ElevenLabs voices.

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/elevenlabs/list-voices/
```

**Request:**
```json
{}
```

**Response:**
```json
{
  "voices": [
    {
      "voice_id": "21m00Tcm4TlvDq8ikWAM",
      "name": "Rachel",
      "category": "premade",
      "labels": {
        "accent": "american",
        "gender": "female"
      }
    }
  ]
}
```

**Frontend Example:**
```javascript
async function listVoices(org, apiKey) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/elevenlabs/list-voices/`,
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
  return data.voices;
}
```

---

## List Models

Get all available TTS models.

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/elevenlabs/list-models/
```

**Request:**
```json
{}
```

**Response:**
```json
[
  {
    "model_id": "eleven_multilingual_v2",
    "name": "Eleven Multilingual v2",
    "description": "Our most advanced model..."
  }
]
```

**Frontend Example:**
```javascript
async function listModels(org, apiKey) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/elevenlabs/list-models/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({})
    }
  );

  return await response.json();
}
```

---

## Text-to-Speech

Convert text to speech audio.

**Endpoint:**
```
POST /api/ai-proxy/orgs/{org}/services/elevenlabs/tts/
```

**Path Template:** `/v1/text-to-speech/{voice_id}`

**Required Path Parameters:**
- `voice_id` - The ID of the voice to use (get from List Voices endpoint)

**Request:**
```json
{
  "path_params": {
    "voice_id": "21m00Tcm4TlvDq8ikWAM"
  },
  "body": {
    "text": "Hello, this is a test.",
    "model_id": "eleven_multilingual_v2",
    "voice_settings": {
      "stability": 0.5,
      "similarity_boost": 0.5
    }
  }
}
```

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `text` | string | Yes | The text to convert to speech |
| `model_id` | string | Yes | The TTS model to use |
| `voice_settings` | object | No | Voice customization settings |
| `voice_settings.stability` | number | No | Voice stability (0.0 - 1.0) |
| `voice_settings.similarity_boost` | number | No | Voice similarity boost (0.0 - 1.0) |

**Response:**
- Content-Type: `audio/mpeg`
- Body: Binary audio data (MP3)

**Frontend Example:**
```javascript
async function textToSpeech(org, apiKey, text, voiceId) {
  const response = await fetch(
    `/api/ai-proxy/orgs/${org}/services/elevenlabs/tts/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        path_params: { voice_id: voiceId },
        body: {
          text: text,
          model_id: 'eleven_multilingual_v2',
          voice_settings: {
            stability: 0.5,
            similarity_boost: 0.5
          }
        }
      })
    }
  );

  if (response.ok) {
    const audioBlob = await response.blob();
    const audioUrl = URL.createObjectURL(audioBlob);
    const audio = new Audio(audioUrl);
    audio.play();
    return audioUrl;
  } else {
    const error = await response.json();
    throw new Error(error.detail || 'TTS request failed');
  }
}
```

---

## Complete Workflow Example

This example demonstrates a complete workflow: discovering voices, selecting one, and generating speech.

```javascript
async function elevenLabsWorkflow(org, apiKey) {
  // Step 1: Get available voices
  const voicesResponse = await fetch(
    `/api/ai-proxy/orgs/${org}/services/elevenlabs/list-voices/`,
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
  const voices = voicesData.voices;

  console.log('Available voices:', voices.map(v => v.name));

  // Step 2: Get available models
  const modelsResponse = await fetch(
    `/api/ai-proxy/orgs/${org}/services/elevenlabs/list-models/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({})
    }
  );
  const models = await modelsResponse.json();

  console.log('Available models:', models.map(m => m.model_id));

  // Step 3: Select a voice and model
  const selectedVoice = voices[0];
  const selectedModel = models[0];

  // Step 4: Generate speech
  const ttsResponse = await fetch(
    `/api/ai-proxy/orgs/${org}/services/elevenlabs/tts/`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Api-Key ${apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        path_params: { voice_id: selectedVoice.voice_id },
        body: {
          text: 'Hello! This is a demonstration of the ElevenLabs text-to-speech API.',
          model_id: selectedModel.model_id,
          voice_settings: {
            stability: 0.5,
            similarity_boost: 0.5
          }
        }
      })
    }
  );

  if (ttsResponse.ok) {
    const audioBlob = await ttsResponse.blob();
    const audioUrl = URL.createObjectURL(audioBlob);

    // Play the audio
    const audio = new Audio(audioUrl);
    audio.play();

    console.log('Audio generated successfully!');
    return audioUrl;
  } else {
    const error = await ttsResponse.json();
    console.error('TTS failed:', error);
    throw new Error(error.detail || 'TTS request failed');
  }
}

// Usage
elevenLabsWorkflow('my-org', 'my-api-key')
  .then(audioUrl => console.log('Audio URL:', audioUrl))
  .catch(error => console.error('Workflow failed:', error));
```

---

## Related Pages

- [Overview](overview.md) - Authentication and service discovery
- [Integration Guide](integration.md) - Path templates and dynamic client
- [HeyGen Integration](heygen.md) - Video generation endpoints
- [Error Handling](errors.md) - Error responses and credential setup
