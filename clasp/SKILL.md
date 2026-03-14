---
name: clasp
description: "How to use clasp (Command Line Apps Script Projects) for local development of Google Apps Script. Use this skill when the user mentions clasp, pushing or pulling Apps Script code, local GAS development, .clasp.json configuration, or syncing code between local files and the Apps Script editor."
---

# Clasp — Local Google Apps Script Development

Clasp lets you edit Apps Script projects locally and push/pull to the cloud.

## 1. Installation

```bash
npm install -g @google/clasp
clasp login  # opens browser for OAuth
```

## 2. Project Setup

### Clone existing project

```bash
clasp clone <scriptId>
```

Find the `scriptId` in Apps Script editor → Project Settings → IDs.

### Create new project

```bash
clasp create --title "My Project" --type sheets
```

## 3. Configuration (`.clasp.json`)

```json
{
  "scriptId": "1LtfG_DB...",
  "rootDir": "",
  "scriptExtensions": [".js", ".gs"],
  "htmlExtensions": [".html"],
  "jsonExtensions": [".json"]
}
```

- `scriptExtensions`: local `.js` files are uploaded as `.gs` server files
- `rootDir`: set to a subdirectory if your repo has non-GAS files at root

## 4. Daily Workflow

```bash
clasp push    # upload local → Apps Script
clasp pull    # download Apps Script → local
clasp open    # open the script editor in browser
clasp status  # show files that would be pushed
```

### Push is destructive

`clasp push` **overwrites** the remote project entirely. Always commit to git before pushing.

### Pull overwrites local

`clasp pull` **overwrites** local files with remote content. Commit first.

## 5. File Mapping

| Local | Remote (Apps Script) |
|-------|---------------------|
| `Menu.js` | `Menu.gs` |
| `Feature.js` | `Feature.gs` |
| `FeatureHtml.html` | `FeatureHtml.html` |
| `appsscript.json` | Project manifest |

## 6. Ignoring Files

Create `.claspignore` (same syntax as `.gitignore`):

```
node_modules/**
.git/**
*.json
!appsscript.json
README.md
```

## 7. Common Pitfalls

- **No partial push**: clasp pushes ALL files — you can't push just one file
- **File naming**: avoid dots in filenames (except extension) — `my.util.js` can cause issues
- **appsscript.json required**: must exist locally, contains runtime config and OAuth scopes
- **Auth expires**: run `clasp login` again if pushes fail with auth errors
- **CRLF warnings**: normal on Windows, safe to ignore
