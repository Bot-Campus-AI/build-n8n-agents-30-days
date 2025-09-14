# Guide.md — Cron Triggers: Your Automation Clock (n8n)

⚠️ **Authentication Required**
- **HTTP workflow**: no auth needed (posts to https://httpbin.org).
- **Google Sheets workflow**: requires a **Google OAuth2** credential in n8n. After import, open each Google Sheets node and select your credential.

---

## 🔎 Overview (Power Pattern)
**Cron Trigger ➝ Prepare Payload ➝ Call Target (HTTP/Sheets) ➝ (Optional) Error Handling**

This guide includes **two real workflows** you can import (JSON provided separately):
1. **Cron Ping (HTTP)** — every 5 minutes in IST ➝ POST to `https://httpbin.org/post`.
2. **Cron Daily Log (Sheets)** — once per day in IST ➝ create a sheet (demo) ➝ append a row.

> Timezone used: **Asia/Kolkata**. Adjust if needed.

---

## ✅ Prerequisites
- n8n instance accessible to you.
- For Google Sheets workflow: a working **Google OAuth2** credential (see your Google OAuth guide if needed).
- You can toggle workflows **Active** in n8n (Cron only runs when the workflow is active, not just saved).

---

## 1) Workflow A — Cron Ping (HTTP POST every 5 minutes, IST)

### Flow
```
Cron (5m, IST) ➝ Function “Build Payload” ➝ HTTP Request (POST → httpbin)
```

### Drag & Drop
- Drag **Cron** node → canvas.
- Drag **Function** node (rename to **Build Payload**) → canvas.
- Drag **HTTP Request** node → canvas.
- Connect arrows in that order.

### Node Config (exact)

**Cron**
- **Trigger**: `Every X`  
- **Every**: `5`  
- **Unit**: `Minutes`  
- **Timezone**: `Asia/Kolkata`

**Function** (Build Payload)
```js
const now = Date.now();
const istIso = new Date(now + 19800000).toISOString(); // +5h30m offset
return [{
  json: {
    job: 'cron_ping',
    note: 'Heartbeat from n8n (every 5 min, IST)',
    epoch_ms: now,
    run_time_ist: istIso
  }
}];
```

**HTTP Request**
- **Method**: `POST`
- **URL**: `https://httpbin.org/post`
- **Send**: `JSON`  
  - **JSON/Body**: `={{ $json }}`  (or enable *JSON parameters* and set body to `$json`)
- **Response**: `JSON`
- **Ignore Response Code**: `ON` (keeps cron firing even if httpbin hiccups)

**Expected Result**
- Every 5 minutes n8n posts a small JSON payload to httpbin and gets a debug echo response.

---

## 2) Workflow B — Cron Daily Log to Google Sheets (IST)

### Flow
```
Cron (Daily, IST) ➝ Function “Build Row” ➝ Google Sheets: Create Spreadsheet ➝ Google Sheets: Append Row
```
> For production, replace **Create Spreadsheet** with a fixed **Spreadsheet ID**, so you append to the same sheet daily. The demo creates a fresh sheet so you can run without pre-setup.

### Drag & Drop
- Drag **Cron** node (rename to **Cron (Daily, IST)**).
- Drag **Function** node (rename to **Build Row**).
- Drag **Google Sheets** node (rename to **Google Sheets - Create Spreadsheet**).
- Drag **Google Sheets** node (rename to **Google Sheets - Append Row**).
- Connect:
  - **Cron ➝ Build Row**
  - **Build Row ➝ Create Spreadsheet**
  - **Create Spreadsheet ➝ Append Row**
  - (Optional parallel) **Build Row ➝ Append Row** (if you later hardcode a fixed documentId)

### Node Config (exact)

**Cron (Daily, IST)**
- **Trigger**: `Every X`
- **Every**: `24`
- **Unit**: `Hours`
- **Timezone**: `Asia/Kolkata`
> To run at a specific time (e.g., 09:00 IST), see **Scheduling Examples** below.

**Function** (Build Row)
```js
const now = Date.now();
const istIso = new Date(now + 19800000).toISOString(); // +5:30
return [{
  json: {
    job: 'daily_log',
    note: 'Automated daily log from n8n (IST)',
    run_time_ist: istIso
  }
}];
```

**Google Sheets - Create Spreadsheet**
- **Operation**: `Create Spreadsheet`
- **Title**: `Cron Daily Log (n8n)`
- **Credentials**: *(select your Google Sheets OAuth2)*

**Google Sheets - Append Row**
- **Operation**: `Append`
- **Spreadsheet** (`documentId`):  
  `={{ $node["Google Sheets - Create Spreadsheet"].json["spreadsheetId"] }}`
- **Range**: `Sheet1!A:C`
- **Options → Value Input Mode**: `RAW`
- **Credentials**: *(select your Google Sheets OAuth2)*

**Expected Result**
- On each run, a new spreadsheet is created and one row is appended to `Sheet1` with `job`, `note`, `run_time_ist`.

---

## 🕰 Scheduling Examples (IST)

### A) Run at **09:00 IST daily**
- **Cron** → `Specific Times`
  - **Hours**: `9`
  - **Minutes**: `0`
  - **Timezone**: `Asia/Kolkata`

### B) Run at **09:05 IST on weekdays (Mon–Fri)**
- **Cron** → `Custom` (Cron expression)
  - `5 9 * * 1-5`
  - **Timezone**: `Asia/Kolkata`

### C) Run at **every hour at 15 and 45 minutes**
- **Cron** → `Custom`
  - `15,45 * * * *`
  - **Timezone**: `Asia/Kolkata`

> Always set the **Timezone** field to avoid surprises when your server is in a different region.

---

## 🛡 Reliability & Error Handling (recommended)
- **Continue On Fail** where non-critical (e.g., HTTP status ≠ 200) so cron doesn’t stop.
- **IF** node branching on `{{$json.error}}` or `{{$json.statusCode}}` ➝ retry, wait, or fallback.
- **Wait** node for backoff (e.g., exponential: 5s → 10s → 20s).
- **Alerting**: send failure summaries to Slack/Email/Sheets for visibility.

---

## ✅ Activate the Workflow
Cron triggers fire **only when the workflow is Active**:
- Toggle **Activate** (top-right in n8n).  
- You can always run a single execution via **Execute Workflow** for testing.

---

## Appendix — Import the sample JSONs
Use **Workflows ➝ Import from File/Clipboard** and paste the JSON for:
- `workflow_cron_http_ping.json`
- `workflow_cron_sheets_daily.json`

Then, in the **Google Sheets** nodes, select your OAuth2 credential and **Save**.

---

**End of Guide**
