---
name: google-apps-script
description: "Patterns and best practices for Google Apps Script development. Use this skill whenever the user is building or modifying Google Apps Script projects (.gs files), creating modal dialogs with HtmlService, using google.script.run for frontend-backend communication, caching data on Google Drive, or working with SpreadsheetApp, DriveApp, DocumentApp. Also trigger when the user mentions Apps Script limits, UrlFetchApp, or PropertiesService."
---

# Google Apps Script Patterns

## 1. Project Structure

Apps Script projects use `.gs` (server-side) and `.html` (client-side) files. When developing locally with clasp, use `.js` extensions for server files.

```
Menu.gs          — Global constants, menu setup, onOpen()
Feature.gs       — Backend logic for a feature
FeatureHtml.html — Frontend UI (HTML + CSS + JS)
```

## 2. Modal Dialogs (HtmlService)

Open a full-screen modal from server code:

```javascript
function openMyTool() {
  const html = HtmlService.createTemplateFromFile('MyToolHtml')
    .evaluate()
    .setTitle('My Tool')
    .setWidth(1600)
    .setHeight(1000);
  SpreadsheetApp.getUi().showModalDialog(html, 'My Tool');
}
```

## 3. Frontend ↔ Backend Communication

### Call server from frontend

```javascript
// Frontend (in .html file)
google.script.run
  .withSuccessHandler(result => {
    // result is the return value from the server function
  })
  .withFailureHandler(err => {
    alert("Error: " + err.message);
  })
  .myServerFunction(arg1, arg2);
```

### Rules

- Server functions must be top-level (not inside objects or classes)
- Arguments and return values are serialized as JSON — no functions, Date objects become strings
- Calls are async — use `withSuccessHandler`/`withFailureHandler`
- Multiple calls can run in parallel

## 4. Global Constants

Define shared constants in `Menu.gs` — they're available across all `.gs` files:

```javascript
// Menu.gs
const MISTRAL_API_KEY = PropertiesService.getScriptProperties().getProperty("MISTRAL_API_KEY");
const MY_FOLDER_ID = "1abc...xyz";
```

## 5. API Keys & Secrets

Never hardcode. Use Script Properties:

```javascript
// Store: Script Editor → Project Settings → Script Properties
// Read:
const key = PropertiesService.getScriptProperties().getProperty("MY_API_KEY");
```

## 6. HTTP Requests (UrlFetchApp)

```javascript
const res = UrlFetchApp.fetch(url, {
  method: "post",
  headers: {
    "Authorization": "Bearer " + apiKey,
    "Content-Type": "application/json"
  },
  payload: JSON.stringify(data),
  muteHttpExceptions: true,  // ALWAYS set this
  followRedirects: true
});

const code = res.getResponseCode();
const body = JSON.parse(res.getContentText());
```

Always set `muteHttpExceptions: true` — without it, non-200 responses throw exceptions instead of returning the error body.

## 7. Google Drive Caching

Cache JSON data as files on Drive to persist across sessions:

```javascript
function loadJsonCache(folderId, filename) {
  try {
    const folder = DriveApp.getFolderById(folderId);
    const files = folder.getFilesByName(filename);
    if (files.hasNext()) {
      return JSON.parse(files.next().getBlob().getDataAsString());
    }
  } catch (e) { console.error("Cache load error:", e); }
  return null;
}

function saveJsonCache(folderId, filename, data) {
  try {
    const folder = DriveApp.getFolderById(folderId);
    const files = folder.getFilesByName(filename);
    const content = JSON.stringify(data);
    if (files.hasNext()) {
      files.next().setContent(content);
    } else {
      folder.createFile(filename, content, MimeType.PLAIN_TEXT);
    }
  } catch (e) { console.error("Cache save error:", e); }
}
```

## 8. Spreadsheet Access

```javascript
const ss = SpreadsheetApp.getActiveSpreadsheet();
const sheet = ss.getSheetByName("My Sheet");
const data = sheet.getDataRange().getValues();  // 2D array
const headers = data[0];  // first row
```

## 9. Drive Folder Traversal

```javascript
function findFileByNameRecursively(folder, fileName) {
  const files = folder.getFilesByName(fileName);
  if (files.hasNext()) return files.next();
  const subFolders = folder.getFolders();
  while (subFolders.hasNext()) {
    const found = findFileByNameRecursively(subFolders.next(), fileName);
    if (found) return found;
  }
  return null;
}
```

## 10. Execution Limits

- **6-minute timeout** for regular scripts (30 min for Workspace Enterprise)
- **50 MB** return value size limit for `google.script.run`
- **20 MB** max file size for `DriveApp.createFile()`
- **100 KB** max payload for `PropertiesService`
- Design long operations to be **resumable**: save progress to Drive, check elapsed time, stop gracefully

```javascript
const START = Date.now();
const MAX_MS = 5 * 60 * 1000;  // 5 min safety margin

items.forEach((item, i) => {
  if (Date.now() - START > MAX_MS) {
    saveProgress(i);  // save state to Drive
    throw new Error("Timeout — resume from item " + i);
  }
  // process item
});
```

## 11. Common Pitfalls

- **`getValues()` returns mixed types**: Numbers, Dates, strings, booleans — always convert explicitly
- **`XmlService.parse()` is fragile**: Fails on DTDs, entities, malformed XML — use regex for robust parsing
- **`DriveApp.getFoldersByName()` searches everywhere**: Use `getFolderById()` for specific folders
- **No `console.log` in server code**: Use `Logger.log()` or `console.log()` (V8 runtime only)
- **HTML escaping**: `HtmlService` sanitizes output — use `createTemplateFromFile().evaluate()` not `createHtmlOutput()`
