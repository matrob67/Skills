---
name: mistral-api
description: "How to use the Mistral AI API for embeddings and LLM chat completions. Use this skill whenever the user mentions Mistral API, mistral-embed, mistral-large, text embeddings, or building tools that call the Mistral AI platform. Also trigger when the user wants to compute semantic similarity, generate text with Mistral models, or integrate Mistral into Google Apps Script or Node.js projects."
---

# Mistral AI API

## 1. Authentication

All requests require a Bearer token in the Authorization header:

```
Authorization: Bearer YOUR_MISTRAL_API_KEY
```

In Google Apps Script, store in Script Properties:

```javascript
const key = PropertiesService.getScriptProperties().getProperty("MISTRAL_API_KEY");
```

## 2. Embeddings

**POST** `https://api.mistral.ai/v1/embeddings`

Compute vector embeddings for text using `mistral-embed` (1024-dimension vectors).

```javascript
const res = UrlFetchApp.fetch("https://api.mistral.ai/v1/embeddings", {
  method: "post",
  headers: { "Authorization": "Bearer " + key, "Content-Type": "application/json" },
  payload: JSON.stringify({
    model: "mistral-embed",
    input: ["text to embed"]  // string or array of strings
  }),
  muteHttpExceptions: true
});

const json = JSON.parse(res.getContentText());
// json.data[i].embedding → float array (1024 dims)
```

### Batching

Send multiple texts in one call (up to ~16KB total). Process in batches of 8 for reliability:

```javascript
const BATCH_SIZE = 8;
for (let b = 0; b < texts.length; b += BATCH_SIZE) {
  const batch = texts.slice(b, b + BATCH_SIZE);
  const res = UrlFetchApp.fetch("https://api.mistral.ai/v1/embeddings", {
    method: "post",
    headers: { "Authorization": "Bearer " + key, "Content-Type": "application/json" },
    payload: JSON.stringify({ model: "mistral-embed", input: batch }),
    muteHttpExceptions: true
  });
  const json = JSON.parse(res.getContentText());
  json.data.forEach((item, idx) => {
    // item.embedding is the vector for batch[idx]
  });
  if (b + BATCH_SIZE < texts.length) Utilities.sleep(300);
}
```

### Input text length

Truncate to ~8000 characters per text. Longer inputs may fail or be silently truncated.

### Cosine Similarity

Compare two embedding vectors:

```javascript
function cosineSimilarity(a, b) {
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

Typical thresholds: >0.9 very similar, >0.8 related, >0.75 somewhat related, <0.75 noise.

## 3. Chat Completions (LLM)

**POST** `https://api.mistral.ai/v1/chat/completions`

### Available Models

- `mistral-large-latest` — Most capable, best for analysis and reasoning
- `mistral-small-latest` — Fast, good for simple tasks
- `mistral-medium-latest` — Balance of speed and quality

### Basic Call

```javascript
const res = UrlFetchApp.fetch("https://api.mistral.ai/v1/chat/completions", {
  method: "post",
  headers: { "Authorization": "Bearer " + key, "Content-Type": "application/json" },
  payload: JSON.stringify({
    model: "mistral-large-latest",
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: "Your prompt here" }
    ],
    max_tokens: 300
  }),
  muteHttpExceptions: true
});

if (res.getResponseCode() !== 200) {
  // Handle error
  return "Error: API " + res.getResponseCode();
}
const parsed = JSON.parse(res.getContentText());
const answer = parsed.choices[0].message.content;
```

### Cleaning Markdown from Responses

Mistral often returns markdown formatting. Strip it for plain text display:

```javascript
const clean = answer.replace(/\*\*/g, '');  // remove bold markers
```

## 4. Caching Embeddings on Google Drive

Store computed embeddings as JSON files to avoid recomputing:

```javascript
const CACHE_FILENAME = "_EMBEDDINGS_CACHE.json";

function loadCache(folderId) {
  const folder = DriveApp.getFolderById(folderId);
  const files = folder.getFilesByName(CACHE_FILENAME);
  if (files.hasNext()) {
    return JSON.parse(files.next().getBlob().getDataAsString());
  }
  return {};
}

function saveCache(folderId, data) {
  const folder = DriveApp.getFolderById(folderId);
  const files = folder.getFilesByName(CACHE_FILENAME);
  const content = JSON.stringify(data);
  if (files.hasNext()) {
    files.next().setContent(content);
  } else {
    folder.createFile(CACHE_FILENAME, content, MimeType.PLAIN_TEXT);
  }
}
```

Cache structure: `{ "unique-key": { vector: [...], meta: {...} } }`

## 5. Rate Limiting

- Add `Utilities.sleep(300)` between embedding batches
- Add `Utilities.sleep(500)` between LLM calls in loops
- On 429 errors, use exponential backoff (2s, 4s, 8s)

## 6. Common Pitfalls

- **Input is array for embeddings**: Even for single text, wrap in array: `input: ["text"]`
- **Markdown in responses**: LLM outputs often contain `**bold**` markers — strip them if displaying as plain text
- **Max tokens**: Set `max_tokens` to control response length, otherwise responses can be very long
- **6-minute Apps Script limit**: For large batches, design operations to be resumable with state saved to Drive
