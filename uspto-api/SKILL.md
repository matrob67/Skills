---
name: uspto-api
description: "How to connect to and query the USPTO Open Data Portal API for patent searches. Use this skill whenever the user mentions USPTO, patent search API, patent data, searching patents by assignee/inventor/title, fetching patent claims, or building tools that interact with the US Patent and Trademark Office. Also trigger when the user asks about patent application data, grant documents, or patent family information from the USPTO."
---

# USPTO Open Data Portal API

The USPTO Open Data Portal provides programmatic access to US patent application data including metadata, assignments, continuity chains, and grant documents.

## 1. Getting an API Key

Register at https://developer.uspto.gov/ to get a free API key. No approval process — you get the key immediately after registration.

Store the key securely. In Google Apps Script, use Script Properties:

```javascript
// Store once via Script Editor > Project Settings > Script Properties
// Key: "USPTO_API_KEY"

// Read in code:
const USPTO_API_KEY = PropertiesService.getScriptProperties().getProperty("USPTO_API_KEY");
```

## 2. Authentication

Every request must include the API key in the `x-api-key` header:

```
x-api-key: YOUR_API_KEY
```

## 3. Search Endpoint

**POST** `https://api.uspto.gov/api/v1/patent/applications/search`

The search endpoint only accepts POST requests with a JSON body. GET requests with query parameters do not support field-specific searches properly.

### Request Body Structure

```json
{
  "q": "<query string>",
  "pagination": { "offset": 0, "limit": 100 },
  "sort": [{ "field": "applicationMetaData.filingDate", "order": "Desc" }],
  "fields": [
    "applicationNumberText",
    "applicationMetaData",
    "grantDocumentMetaData",
    "pgpubDocumentMetaData",
    "parentContinuityBag",
    "foreignPriorityBag",
    "assignmentBag"
  ]
}
```

### Query Syntax (`q` field)

Field-specific queries use the format `fieldPath:"value"`. Boolean operators `AND`, `OR` are supported.

```
// Search by assignee
assignmentBag.assigneeBag.assigneeNameText:"Anthropic"

// Multiple assignee names
assignmentBag.assigneeBag.assigneeNameText:"OpenAI" OR assignmentBag.assigneeBag.assigneeNameText:"Open AI"

// Search by inventor
applicationMetaData.firstInventorName:"John Smith"

// Search by title keyword
applicationMetaData.inventionTitle:"neural network"
```

The `assignmentBag` query returns any patent where the name appears anywhere in the assignment history — this includes current owners, previous owners who transferred the patent, and security interest holders (pledges). It does not distinguish between these roles.

### Pagination

- `limit` maximum is **100**. Requesting more returns a 400 error.
- Loop with `offset += 100` until `offset >= totalCount`.
- The response includes a `count` field with the total number of matching results.

```javascript
let offset = 0;
const pageSize = 100;
let totalCount = 0;

do {
  const payload = {
    q: query,
    pagination: { offset: offset, limit: pageSize },
    sort: [{ field: "applicationMetaData.filingDate", order: "Desc" }],
    fields: [/* ... */]
  };

  const res = UrlFetchApp.fetch(URL, {
    method: "post",
    headers: { "x-api-key": apiKey, "Content-Type": "application/json" },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  });

  const json = JSON.parse(res.getContentText());
  totalCount = json.count || 0;

  (json.patentFileWrapperDataBag || []).forEach(raw => {
    // process each patent
  });

  offset += pageSize;
  if (offset < totalCount) Utilities.sleep(500); // rate limiting
} while (offset < totalCount);
```

### Rate Limiting

The API returns **429 Too Many Requests** when rate-limited. Handle with exponential backoff:

```javascript
function fetchWithRetry_(url, options, maxRetries) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const res = UrlFetchApp.fetch(url, options);
    if (res.getResponseCode() === 429 && attempt < maxRetries) {
      Utilities.sleep(2000 * (attempt + 1));
      continue;
    }
    return res;
  }
}
```

Add `Utilities.sleep(500)` between pagination requests as a baseline courtesy.

## 4. Response Data Structure

Each item in `patentFileWrapperDataBag` contains:

### `applicationMetaData` — Core Patent Info

| Field | Description | Example |
|-------|-------------|---------|
| `inventionTitle` | Title of the invention | "Neural Network Training System" |
| `filingDate` | Application filing date | "2024-10-08" |
| `grantDate` | Grant date (null if not granted) | "2026-03-03" |
| `patentNumber` | US patent number (null if pending) | "12566913" |
| `applicationStatusDescriptionText` | Current status | "Patented Case" |
| `firstInventorName` | Lead inventor | "John SMITH" |
| `applicationTypeCode` | Type code | "UTL", "PRV", "PCT" |
| `applicationTypeLabelName` | Type label | "Utility", "Provisional" |
| `earliestPublicationNumber` | Published application number | "US20250298495A1" |
| `pctPublicationNumber` | PCT publication number | "WO2025199330" |

### `grantDocumentMetaData` — Grant XML Access

Present only for granted patents. The key field is `fileLocationURI` which points to the full XML grant document.

```javascript
const xmlUrl = raw.grantDocumentMetaData
  ? raw.grantDocumentMetaData.fileLocationURI
  : null;
```

### `parentContinuityBag` — Patent Family Chain

Array of parent applications (continuations, divisionals, CIPs):

```javascript
raw.parentContinuityBag.forEach(p => {
  p.parentApplicationNumberText  // e.g., "17123456"
  p.parentApplicationfilingDate  // e.g., "2022-05-15" (note: lowercase 'f')
  p.claimParentageTypeCodeDescriptionText  // e.g., "Continuation"
});
```

### `foreignPriorityBag` — Foreign Priority Claims

```javascript
raw.foreignPriorityBag.forEach(fp => {
  fp.applicationNumberText  // foreign app number
  fp.filingDate             // priority date
});
```

### `assignmentBag` — Assignment History

This is an **array** of assignment entries (not a nested object). Each entry records a transfer, assignment, or security agreement:

```javascript
const entries = [].concat(raw.assignmentBag || []);
entries.forEach(entry => {
  // Who received the patent
  const assignees = [].concat(entry.assigneeBag || []);
  assignees.forEach(a => {
    a.assigneeNameText  // e.g., "ANTHROPIC, PBC"
  });

  // Who transferred the patent
  const assignors = [].concat(entry.assignorBag || []);
  assignors.forEach(a => {
    a.assignorName  // e.g., "ADEPT AL LABS INC." (note: assignorName, not assignorNameText)
  });

  entry.conveyanceText          // e.g., "ASSIGNMENT OF ASSIGNOR'S INTEREST"
  entry.assignmentRecordedDate  // e.g., "2025-04-09"
  entry.reelAndFrameNumber      // e.g., "070785/0387"
});
```

**Important nuances:**

- The assignor field is `assignorName` (not `assignorNameText` like the assignee field)
- `conveyanceText` containing "SECURITY" indicates a pledge/collateral agreement, not a real ownership transfer (e.g., Morgan Stanley as collateral agent)
- The same patent can have multiple assignment entries showing the ownership chain over time
- Querying by assignee name returns patents where that name appears *anywhere* in the assignment history, including as a previous owner or security holder

## 5. Extracting Claims from Grant XML

The `fileLocationURI` from `grantDocumentMetaData` returns a full XML document. Fetch it with the same API key:

```javascript
const res = fetchWithRetry_(xmlUrl, {
  method: "get",
  headers: { "x-api-key": apiKey },
  muteHttpExceptions: true,
  followRedirects: true
}, 3);
```

**Do not use `XmlService.parse()`** — USPTO XML contains DTD references and entities that cause Apps Script's XmlService to fail. Use regex instead:

```javascript
const xmlText = res.getContentText();
const claims = [];
const claimPattern = /<claim\s+id="([^"]+)"[^>]*num="(\d+)"[^>]*>([\s\S]*?)<\/claim>/g;
let match;

while ((match = claimPattern.exec(xmlText)) !== null) {
  const claimNum = parseInt(match[2], 10);
  const claimBody = match[3];

  // Strip XML tags to get plain text
  const text = claimBody.replace(/<[^>]+>/g, " ").replace(/\s+/g, " ").trim();

  // Independent claims have no <claim-ref> element (no dependency on another claim)
  const isIndependent = !/<claim-ref/.test(claimBody);

  claims.push({ num: claimNum, text: text, independent: isIndependent });
}
```

## 6. Building Google Patents URLs

Construct links to Google Patents based on available identifiers:

```javascript
function buildGooglePatentsUrl(meta) {
  if (meta.patentNumber) return "https://patents.google.com/patent/US" + meta.patentNumber;
  if (meta.earliestPublicationNumber) return "https://patents.google.com/patent/" + meta.earliestPublicationNumber;
  if (meta.pctPublicationNumber) return "https://patents.google.com/patent/" + meta.pctPublicationNumber;
  return null;
}
```

Very recent patents (days/weeks old) may not yet appear on Google Patents.

## 7. Google Apps Script Specifics

- Use `UrlFetchApp.fetch()` for HTTP requests
- Always set `muteHttpExceptions: true` to handle errors gracefully
- Apps Script execution limit is **6 minutes** — design batch operations to be resumable
- Store API keys in `PropertiesService.getScriptProperties()`, never hardcoded
- Use `Utilities.sleep(ms)` for rate limiting between requests
- `JSON.parse(res.getContentText())` to parse responses
- For large data sets, cache results in Google Drive as JSON files using `DriveApp`

## 8. Common Pitfalls

- **GET vs POST**: The search endpoint requires POST with JSON body. GET with query params does not support field-specific queries.
- **Pagination limit**: Max 100 per page. Requesting 200 returns HTTP 400.
- **XmlService**: Breaks on USPTO XML. Use regex for claims extraction.
- **assignorName vs assignorNameText**: Assignors use `assignorName`, assignees use `assigneeNameText`.
- **assignmentBag is an array**: Not a nested object with an `assignment` property. Use `[].concat(raw.assignmentBag || [])`.
- **Assignment queries are broad**: Searching by assignee name returns patents where that name appears in any role (current owner, previous owner, security holder).
