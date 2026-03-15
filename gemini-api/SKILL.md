---
name: gemini-api
description: "How to connect to and use the Google Gemini API for text generation, chat, and multimodal AI tasks. Use this skill whenever the user mentions Gemini API, Gemini models, Google AI Studio, generativelanguage.googleapis.com, or wants to call Gemini from code. Also trigger when the user wants to use gemini-2.5-flash, gemini-2.5-pro, gemini-3-flash, gemini-3-pro, gemini-3.1-pro, or any Google Gemini model for text generation, analysis, chat completions, image understanding, or AI-powered features. Trigger when the user asks about connecting to Google AI, using a Google API key with Gemini, or building apps with Gemini models."
---

# Google Gemini API

The Gemini API provides access to Google's latest AI models for text generation, chat, multimodal understanding, and more. Authentication is via API key (no OAuth needed).

## 1. Authentication

Get an API key from [Google AI Studio](https://aistudio.google.com/apikey) or use an existing Google Cloud API key with the Generative Language API enabled.

The API key is passed as a query parameter `?key=YOUR_API_KEY` on every request.

**Base URL:** `https://generativelanguage.googleapis.com/v1beta`

## 2. Available Models

| Model | ID | Best for |
|-------|-----|----------|
| Gemini 3.1 Pro | `gemini-3.1-pro-preview` | Most capable, complex reasoning |
| Gemini 3 Pro | `gemini-3-pro-preview` | Strong reasoning, balanced |
| Gemini 3 Flash | `gemini-3-flash-preview` | Fast, good quality |
| Gemini 2.5 Pro | `gemini-2.5-pro` | Stable, production-ready |
| Gemini 2.5 Flash | `gemini-2.5-flash` | Fast, cheap, stable |
| Gemini 2.0 Flash | `gemini-2.0-flash` | Legacy fast model |

All models support up to 1M input tokens and 65K output tokens.

### List models
```
GET /v1beta/models?key=YOUR_API_KEY
```

## 3. Generate Content (Text)

### Simple text generation

```
POST /v1beta/models/{model}:generateContent?key=YOUR_API_KEY
Content-Type: application/json

{
  "contents": [
    {
      "parts": [{"text": "Your prompt here"}]
    }
  ]
}
```

**Response:**
```json
{
  "candidates": [
    {
      "content": {
        "parts": [{"text": "The model's response"}],
        "role": "model"
      },
      "finishReason": "STOP"
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 10,
    "candidatesTokenCount": 50,
    "totalTokenCount": 60
  }
}
```

### With generation config

```json
{
  "contents": [{"parts": [{"text": "prompt"}]}],
  "generationConfig": {
    "temperature": 0.7,
    "topP": 0.95,
    "topK": 40,
    "maxOutputTokens": 8192
  }
}
```

### With system instruction

```json
{
  "contents": [{"parts": [{"text": "user message"}]}],
  "systemInstruction": {
    "parts": [{"text": "You are a helpful assistant specialized in..."}]
  }
}
```

## 4. Multi-turn Chat

Send the full conversation history in `contents`:

```json
{
  "contents": [
    {"role": "user", "parts": [{"text": "Hello"}]},
    {"role": "model", "parts": [{"text": "Hi! How can I help?"}]},
    {"role": "user", "parts": [{"text": "What is OSINT?"}]}
  ]
}
```

## 5. Code Examples

### Node.js (https module, no dependencies)

```javascript
const https = require('https');

function callGemini(apiKey, prompt, model = 'gemini-2.5-flash') {
  return new Promise((resolve, reject) => {
    const postData = JSON.stringify({
      contents: [{ parts: [{ text: prompt }] }]
    });

    const req = https.request({
      hostname: 'generativelanguage.googleapis.com',
      path: `/v1beta/models/${model}:generateContent?key=${apiKey}`,
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Content-Length': Buffer.byteLength(postData),
      },
    }, (res) => {
      let body = '';
      res.on('data', (chunk) => body += chunk);
      res.on('end', () => {
        const data = JSON.parse(body);
        if (data.candidates?.[0]) {
          resolve(data.candidates[0].content.parts[0].text);
        } else {
          reject(new Error(data.error?.message || 'No response'));
        }
      });
    });

    req.on('error', reject);
    req.write(postData);
    req.end();
  });
}
```

### Python (requests)

```python
import requests

def call_gemini(api_key, prompt, model="gemini-2.5-flash"):
    url = f"https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={api_key}"
    response = requests.post(url, json={
        "contents": [{"parts": [{"text": prompt}]}]
    })
    data = response.json()
    return data["candidates"][0]["content"]["parts"][0]["text"]
```

### curl

```bash
curl -X POST "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"contents": [{"parts": [{"text": "Hello"}]}]}'
```

## 6. Streaming

For streaming responses, use `streamGenerateContent`:

```
POST /v1beta/models/{model}:streamGenerateContent?key=YOUR_API_KEY&alt=sse
```

Returns Server-Sent Events (SSE) with partial responses.

## 7. Count Tokens

```
POST /v1beta/models/{model}:countTokens?key=YOUR_API_KEY
Content-Type: application/json

{
  "contents": [{"parts": [{"text": "Your text here"}]}]
}
```

## 8. Safety Settings

Override default safety filters:

```json
{
  "contents": [{"parts": [{"text": "prompt"}]}],
  "safetySettings": [
    {"category": "HARM_CATEGORY_HARASSMENT", "threshold": "BLOCK_ONLY_HIGH"},
    {"category": "HARM_CATEGORY_HATE_SPEECH", "threshold": "BLOCK_ONLY_HIGH"},
    {"category": "HARM_CATEGORY_SEXUALLY_EXPLICIT", "threshold": "BLOCK_ONLY_HIGH"},
    {"category": "HARM_CATEGORY_DANGEROUS_CONTENT", "threshold": "BLOCK_ONLY_HIGH"}
  ]
}
```

## 9. Error Handling

| HTTP Code | Meaning |
|-----------|---------|
| 200 | Success |
| 400 | Invalid request (bad JSON, missing fields) |
| 403 | API key invalid or quota exceeded |
| 404 | Model not found |
| 429 | Rate limit exceeded — wait and retry |
| 500 | Server error — retry |

## 10. Rate Limits & Pricing

- Free tier: 15 RPM (requests per minute) for most models
- Paid tier: much higher limits
- Gemini Flash models are significantly cheaper than Pro models
- Check current pricing at ai.google.dev/pricing
