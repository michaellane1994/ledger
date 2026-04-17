---
name: google-sheets-backend
description: How to set up a static website (single HTML file or GitHub Pages) that uses Google Sheets as a database via an Apps Script web app. Covers the full stack: Apps Script REST API, push/pull sync, localStorage caching, and deployment.
user-invocable: false
---

# Setting Up Google Sheets as a Backend for a Static Site

## Overview

The pattern: a self-contained HTML file runs entirely in the browser. All data is stored locally in `localStorage` for instant load with no server. Google Sheets acts as a shared database — a Google Apps Script web app exposes a simple REST API (POST only, no auth needed beyond the URL) that reads and writes the sheet. Any device can push its local data up or pull the latest data down.

This is ideal for:
- Personal tools shared across devices
- Small household or team apps
- Anything that needs persistence without a real backend

---

## Architecture

```
Browser (localStorage)
       |
       |  POST { action: 'push' | 'pull', ...data }
       v
Google Apps Script Web App  (doPost)
       |
       v
Google Spreadsheet  (one tab per data type)
```

- **Push** = overwrite entire sheet with local data (never append)
- **Pull** = replace all local data with sheet data (never merge)
- No incremental sync — always full replace in both directions

---

## Step 1 — Create the Google Sheet

1. Go to [sheets.google.com](https://sheets.google.com) and create a new spreadsheet
2. Name it (e.g. "My App Data")
3. You do not need to create tabs manually — the Apps Script creates them on first push

---

## Step 2 — Create the Apps Script

1. In the sheet: **Extensions → Apps Script**
2. Delete any existing code and paste the following template:

```js
const DATA_SHEET = 'Items'; // rename to match your data

function doPost(e) {
  const lock = LockService.getScriptLock();
  lock.tryLock(10000);
  try {
    const data = JSON.parse(e.postData.contents);
    const ss = SpreadsheetApp.getActiveSpreadsheet();

    if (data.action === 'push') {
      writeSheet(ss, DATA_SHEET,
        ['id', 'date', 'name', 'amount', 'category', 'note'],
        (data.items || []).map(item => [
          item.id, item.date, item.name, item.amount || 0,
          item.category || '', item.note || ''
        ])
      );
      return buildResponse({ ok: true, count: (data.items || []).length });
    }

    if (data.action === 'pull') {
      const items = readSheet(ss, DATA_SHEET, r => ({
        id: r.id,
        date: r.date,
        name: r.name,
        amount: parseFloat(r.amount) || 0,
        category: r.category,
        note: r.note
      }));
      return buildResponse({ ok: true, items });
    }

    return buildResponse({ ok: false, error: 'Unknown action' });
  } catch (err) {
    return buildResponse({ ok: false, error: err.toString() });
  } finally {
    lock.releaseLock();
  }
}

function writeSheet(ss, name, headers, rows) {
  let sheet = ss.getSheetByName(name);
  if (!sheet) sheet = ss.insertSheet(name);
  sheet.clearContents();
  const data = rows.length ? [headers, ...rows] : [headers];
  sheet.getRange(1, 1, data.length, headers.length).setValues(data);
}

function readSheet(ss, name, mapper) {
  const sheet = ss.getSheetByName(name);
  if (!sheet) return [];
  const rows = sheet.getDataRange().getValues();
  if (rows.length < 2) return [];
  const headers = rows[0];
  return rows.slice(1).map(row => {
    const r = {};
    headers.forEach((h, i) => r[h] = row[i]);
    return mapper(r);
  });
}

function buildResponse(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

3. Adapt the columns in `DATA_SHEET` to match your data structure
4. Add additional `writeSheet` / `readSheet` calls for each data type (e.g. one sheet per table)

---

## Step 3 — Deploy as a Web App

1. Click **Deploy → New deployment**
2. Type: **Web app**
3. Execute as: **Me**
4. Who has access: **Anyone** (this makes the URL the only security — keep it private)
5. Click **Deploy**
6. Copy the web app URL — it looks like:
   `https://script.google.com/macros/s/AKfycb.../exec`

**Important:** Every time you change the Apps Script code, you must deploy a **new version**:
Deploy → Manage deployments → edit (pencil icon) → Version: New version → Deploy

---

## Step 4 — Client-Side JavaScript

### Store the URL

```js
let sheetsUrl = localStorage.getItem('myapp_sheets_url') || '';

function saveSheetsUrl(url) {
  sheetsUrl = url.trim();
  localStorage.setItem('myapp_sheets_url', sheetsUrl);
}
```

Expose a settings input so the user can paste their URL in once per device.

### Push function

```js
async function pushData() {
  if (!sheetsUrl) return;
  const payload = {
    action: 'push',
    items: items,            // your local data array
    // add more arrays here
  };
  try {
    const res = await fetch(sheetsUrl, {
      method: 'POST',
      body: JSON.stringify(payload)
    });
    const data = await res.json();
    if (!data.ok) throw new Error(data.error);
    console.log('Pushed', data.count, 'items');
  } catch (e) {
    console.error('Push failed:', e);
  }
}
```

### Pull function

```js
async function pullData() {
  if (!sheetsUrl) return;
  if (!confirm('Pull will replace ALL local data. Continue?')) return;
  try {
    const res = await fetch(sheetsUrl, {
      method: 'POST',
      body: JSON.stringify({ action: 'pull' })
    });
    const data = await res.json();
    if (!data.ok) throw new Error(data.error);
    items = data.items || [];
    localStorage.setItem('myapp_items', JSON.stringify(items));
    renderApp();
  } catch (e) {
    alert('Pull failed: ' + e.message);
  }
}
```

### Auto-push after data changes

Call `pushData()` (without await, let it run in background) after every write operation:

```js
function addItem(item) {
  items.push(item);
  saveLocal();
  renderApp();
  pushData(); // background sync
}
```

---

## Step 5 — localStorage Persistence

Always save to localStorage immediately — this means the app works offline and loads instantly:

```js
const PREFIX = 'myapp_'; // use a unique prefix to avoid collisions

function saveLocal() {
  localStorage.setItem(PREFIX + 'items', JSON.stringify(items));
}

function loadLocal() {
  const raw = localStorage.getItem(PREFIX + 'items');
  return raw ? JSON.parse(raw) : [];
}

let items = loadLocal(); // initialise from localStorage on page load
```

---

## Step 6 — Deploy the HTML (GitHub Pages)

1. Create a GitHub repository
2. Add your HTML file as `index.html` in the root
3. Go to repository Settings → Pages → Source: Deploy from branch → main / root
4. Your site is live at `https://yourusername.github.io/reponame`

To update: `git add . && git commit -m "description" && git push`
GitHub Pages republishes automatically within ~30 seconds.

---

## Key Rules

- **Push always overwrites** — never append to the sheet
- **Pull always fully replaces** — never merge with local data; always confirm before pulling
- **One URL per deployment** — if you redeploy, the URL changes; update it in Settings on every device
- **Schema changes require redeployment** — if you add/remove columns, update the Apps Script and deploy a new version, then Push to rewrite the sheet headers
- **CORS on mobile Safari** — fetch can occasionally fail with "Load failed" on iOS; usually resolves on retry

---

## Multiple Data Types

Add one sheet per data type. In the Apps Script, add a constant and a write/read call for each:

```js
const ITEMS_SHEET   = 'Items';
const SETTINGS_SHEET = 'Settings';
const LOG_SHEET     = 'Log';

// In push:
writeSheet(ss, ITEMS_SHEET,    [...], data.items.map(...));
writeSheet(ss, SETTINGS_SHEET, [...], data.settings.map(...));
writeSheet(ss, LOG_SHEET,      [...], data.log.map(...));

// In pull:
const items    = readSheet(ss, ITEMS_SHEET,    r => ({...}));
const settings = readSheet(ss, SETTINGS_SHEET, r => ({...}));
const log      = readSheet(ss, LOG_SHEET,      r => ({...}));
return buildResponse({ ok: true, items, settings, log });
```

The `writeSheet` function creates the tab automatically if it does not exist.
