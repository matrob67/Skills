---
name: euipo-caselaw
description: "How to connect to and query the EUIPO eSearch Case Law API for European trademark and design decisions. Use this skill whenever the user mentions EUIPO, eSearch Case Law, European trademark decisions, EUTM opposition or cancellation decisions, EU trademark conflicts, likelihood of confusion cases, Boards of Appeal decisions, or wants to search, scrape, download, or query the EUIPO case law database. Also trigger when the user wants to analyze EU trademark conflict outcomes, build prediction models from EUIPO data, or download decision PDFs from the European IP office."
---

# EUIPO eSearch Case Law API

The EUIPO (European Union Intellectual Property Office) eSearch Case Law database contains ~342,000 decisions and is accessible via an undocumented but stable JSON API. No API key or authentication is required.

## 1. Endpoints Overview

**Base URL:** `https://euipo.europa.eu`

### Search Endpoints (POST, JSON body)

| Type | Endpoint |
|------|----------|
| Trade mark decisions | `/caselaw/officesearch/json/{lang}` |
| Design decisions | `/caselaw/rcdsearch/json/{lang}` |
| National court judgments | `/caselaw/nationalsearch/json/{lang}` |
| Preliminary rulings | `/caselaw/preliminary/json/{lang}` |

`{lang}` = `en`, `fr`, `de`, `es`, `it`, etc.

### Other Endpoints

| Purpose | Method | Endpoint |
|---------|--------|----------|
| Configuration (all dropdown values, URLs) | GET | `/caselaw/conf/all` |
| PDF export | GET | `/caselaw/officesearch/resultspdf/{lang}/?query={query}` |
| RSS feed (office) | GET | `/caselaw/officesearch/rss` |
| RSS feed (national) | GET | `/caselaw/nationalsearch/rss` |

### Autocomplete Endpoints (GET)

| Data | Endpoint |
|------|----------|
| Application names | `/caselaw/application/search/autocomplete?word={word}&rows={rows}` |
| Legal norms (TM) | `/caselaw/norms/search/autocomplete?word={word}` |
| Legal norms (Design) | `/caselaw/norms/rcd/search/autocomplete?word={word}` |
| BoA keywords (TM) | `/caselaw/boakeywords/ctm/search/autocomplete/{lang}?word={word}&rows={rows}` |
| GC keywords (TM) | `/caselaw/gckeywords/ctm/search/autocomplete/{lang}?word={word}&rows={rows}` |
| Persons | `/caselaw/person/autocomplete?word={word}&rows={rows}&inputName={inputName}` |
| Rapporteur | `/caselaw/rapporteur/autocomplete?word={word}&rows={rows}` |
| Court names | `/caselaw/nac/courtName/search/autocomplete/{lang}?word={word}&rows={rows}` |

## 2. Search Request Format

All search endpoints accept POST with a JSON body:

```json
{
  "criteria": [
    {
      "criterion": "DecisionDate",
      "term": "*",
      "operator": "AND",
      "condition": "contains",
      "group": 1
    }
  ],
  "sort": {
    "field": "DecisionDate",
    "order": "desc"
  },
  "start": 0,
  "resultsPerPage": 100
}
```

### Criteria Fields

| Criterion | Description | Example term | operator | condition |
|-----------|-------------|-------------|----------|-----------|
| `DecisionDate` | Wildcard search all dates | `*` | `"AND"` | `"contains"` |
| `DecisionDateFrom` | Start of date range (DD/MM/YYYY) | `"01/01/2020"` | `null` | `null` |
| `DecisionDateTo` | End of date range (DD/MM/YYYY) | `"31/12/2025"` | `null` | `null` |
| `MarkVerbalElementText` | Trademark name/word element | `"APPLE"` | `"AND"` | `"contains"` |
| `ApplicationNumber` | TM/design registration number | `"018834813"` | `"AND"` | `"contains"` |
| `DecisionCaseReference` | Case number | `"003199099"` | `"AND"` | `"contains"` |
| `ClassNumber` | Nice classification number | `"9"` | `"AND"` | `"is"` |
| `DecisionLanguage` | Language code | `"en"` | `"AND"` | `"is"` |
| `ApplicantName` | TM owner name | `"Samsung"` | `"AND"` | `"contains"` |
| `OpponentName` | Opponent name | `"Apple Inc"` | `"AND"` | `"contains"` |
| `attr_fulltext` | Full-text search in decisions | `"likelihood of confusion"` | `"AND"` | `"contains"` |

**Date criteria are special**: `DecisionDateFrom` and `DecisionDateTo` require `"operator": null` and `"condition": null`. All other criteria use string operators.

### Multiple Criteria

Combine multiple criteria in the array. They are joined by the `operator` field (AND/OR):

```json
{
  "criteria": [
    {"criterion": "DecisionDateFrom", "term": "01/01/2020", "operator": null, "condition": null, "group": 1},
    {"criterion": "DecisionDateTo", "term": "31/12/2025", "operator": null, "condition": null, "group": 1},
    {"criterion": "MarkVerbalElementText", "term": "NIKE", "operator": "AND", "condition": "contains", "group": 1}
  ],
  "sort": {"field": "DecisionDate", "order": "desc"},
  "start": 0,
  "resultsPerPage": 100
}
```

### Pagination

- `resultsPerPage`: max 100
- `start`: offset (0-based)
- Loop with `start += 100` until `start >= numFound`
- Add a delay of 300-500ms between requests to avoid rate limiting

## 3. Response Format

```json
{
  "errorLabel": null,
  "results": [ ... ],
  "numFound": 342195
}
```

### Result Object (Trade Mark Decision)

```json
{
  "type": "OPPOSITION",
  "typeLabel": "Opposition",
  "date": "13/03/2026",
  "uniqueSolrKey": "OPP_20260313_003199099_018834813",
  "url": "https://euipo.europa.eu/eSearchCLW/#key/trademark/OPP_20260313_003199099_018834813",
  "languagesOriginal": [
    {
      "code": "en",
      "label": "English",
      "pdfUrl": "https://euipo.europa.eu/copla/trademark/data/018834813/download/CLW/OPP/2026/EN/20260313_003199099.doc?app=caselaw&casenum=003199099&trTypeDoc=NA"
    }
  ],
  "languagesHumanTranslated": [],
  "languagesMachineTranslated": [],
  "caseNumber": "003199099",
  "ipRight": "EUTM",
  "outcome": "Partially refused EUTMA/IR",
  "appealed": "No",
  "entityNumber": "018834813",
  "entityStatus": "Application opposed",
  "norms": ["Article 8(1)(b) EUTMR"],
  "entityImage": "/copla/image/...",
  "entityName": "I INTER PROTECCION",
  "entityType": "Figurative",
  "opposedClasses": ["9", "36", "45"],
  "earlierName": "INTER",
  "earlierType": "Word"
}
```

### Decision Types (type | ipRight)

| type | ipRight | Description |
|------|---------|-------------|
| `OPPOSITION` | `EUTM` | Trademark opposition decision |
| `CANCELLATION` | `EUTM` | Trademark cancellation/invalidity |
| `EXAMINATION` | `EUTM` | Absolute grounds refusal |
| `EXAMINATION` | `IR designating the EU` | International registration exam |
| `APPEAL` | `EUTM` | Board of Appeal decision |
| `INVALIDITY` | `RCD` | Design invalidity |

### Outcome Values (for Opposition)

Common `outcome` values include:
- `"Opposition rejected"` — applicant wins
- `"Totally refused EUTMA/IR"` — opponent wins (total refusal)
- `"Partially refused EUTMA/IR"` — partial win for opponent
- `"Opposition deemed not entered"` — procedural
- `"Cancellation rejected"` / `"Cancellation totally upheld..."` — for cancellation type

### Key Fields for ML/Prediction

| Field | Role | Notes |
|-------|------|-------|
| `outcome` | **Target variable (Y)** | What the decision was |
| `norms` | Legal basis | Articles applied (e.g., Art. 8(1)(b) EUTMR = likelihood of confusion) |
| `entityName` + `entityType` | Contested mark | Name and type (Word, Figurative, etc.) |
| `earlierName` + `earlierType` | Earlier/opposing mark | The mark cited against |
| `opposedClasses` | Nice classes | Product/service classes in conflict |
| `pdfUrl` | Full decision text | `.doc` file with detailed reasoning |

## 4. Downloading Decision Documents

Each decision has a `pdfUrl` (actually a `.doc` file) in `languagesOriginal[0].pdfUrl`. Download with a simple GET request — no authentication needed:

```javascript
// Node.js
const https = require('https');
const fs = require('fs');

function downloadDecision(pdfUrl, outputPath) {
  return new Promise((resolve, reject) => {
    const file = fs.createWriteStream(outputPath);
    https.get(pdfUrl, (res) => {
      if (res.statusCode === 301 || res.statusCode === 302) {
        // Follow redirect
        https.get(res.headers.location, (res2) => {
          res2.pipe(file);
          file.on('finish', () => { file.close(); resolve(); });
        });
        return;
      }
      res.pipe(file);
      file.on('finish', () => { file.close(); resolve(); });
    }).on('error', reject);
  });
}
```

```javascript
// Google Apps Script
function downloadDecision(pdfUrl) {
  const res = UrlFetchApp.fetch(pdfUrl, { muteHttpExceptions: true, followRedirects: true });
  if (res.getResponseCode() === 200) {
    return res.getBlob(); // Can save to Drive with DriveApp
  }
  return null;
}
```

## 5. Volume Statistics

| Year | Decisions (all types) |
|------|-----------------------|
| 2016 | 16,061 |
| 2017 | 17,909 |
| 2018 | 16,398 |
| 2019 | 17,959 |
| 2020 | 17,120 |
| 2021 | 17,223 |
| 2022 | 19,582 |
| 2023 | 19,672 |
| 2024 | 19,526 |
| 2025 | 19,402 |

~64% of all decisions are EUTM Opposition + Cancellation (~220k over 10 years).

## 6. Bulk Scraping Pattern

For large-scale data collection, paginate through all results with a wildcard search:

```javascript
const BATCH_SIZE = 100;
const DELAY_MS = 500;

async function scrapeAll() {
  let start = 0;
  let total = Infinity;

  while (start < total) {
    const payload = {
      criteria: [
        { criterion: "DecisionDate", term: "*", operator: "AND", condition: "contains", group: 1 }
      ],
      sort: { field: "DecisionDate", order: "desc" },
      start: start,
      resultsPerPage: BATCH_SIZE
    };

    const response = await postJSON(payload);
    total = response.numFound;

    const relevant = response.results.filter(r =>
      (r.type === 'OPPOSITION' || r.type === 'CANCELLATION') &&
      r.ipRight?.includes('EUTM')
    );

    // Process relevant results...

    start += BATCH_SIZE;
    await sleep(DELAY_MS);
  }
}
```

To filter by date range, replace the wildcard criterion with `DecisionDateFrom` / `DecisionDateTo` (remember: `operator: null`, `condition: null`).

## 7. Google Apps Script Integration

```javascript
function searchEUIPO(markName, dateFrom, dateTo) {
  const criteria = [];

  if (markName) {
    criteria.push({
      criterion: "MarkVerbalElementText",
      term: markName,
      operator: "AND",
      condition: "contains",
      group: 1
    });
  }

  if (dateFrom) {
    criteria.push({ criterion: "DecisionDateFrom", term: dateFrom, operator: null, condition: null, group: 1 });
  }
  if (dateTo) {
    criteria.push({ criterion: "DecisionDateTo", term: dateTo, operator: null, condition: null, group: 1 });
  }

  // Default: wildcard if no criteria
  if (criteria.length === 0) {
    criteria.push({ criterion: "DecisionDate", term: "*", operator: "AND", condition: "contains", group: 1 });
  }

  const payload = {
    criteria: criteria,
    sort: { field: "DecisionDate", order: "desc" },
    start: 0,
    resultsPerPage: 100
  };

  const res = UrlFetchApp.fetch("https://euipo.europa.eu/caselaw/officesearch/json/en", {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  });

  return JSON.parse(res.getContentText());
}
```

## 8. Common Pitfalls

- **Date criteria must use `null` operator/condition**: Using `"AND"` / `"contains"` with `DecisionDateFrom` or `DecisionDateTo` returns an API error.
- **Date format is DD/MM/YYYY**: Not YYYY-MM-DD. Response dates also use DD/MM/YYYY.
- **No authentication required**: The API is public, but be respectful with request rates (300-500ms delay).
- **pdfUrl points to .doc files**: Despite the field name, decision documents are Word `.doc` files, not PDFs.
- **Results may include non-EUTM**: A wildcard search returns all IP types (EUTM, RCD, IR). Filter by `ipRight` containing `"EUTM"` for trademark-only results.
- **Max 100 results per page**: The API silently caps at 100 even if you request more.
- **Empty criteria array causes 500 error**: Always include at least one criterion (use the `DecisionDate: *` wildcard as a catch-all).
