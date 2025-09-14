# Example B — Daily KPI at **09:00 IST** → Google Sheets (n8n)

**Flow:**  
**Schedule Trigger (09:00 IST)** → **Function (Build Row)** → **Google Sheets (Append Row)**

---

## ✅ Prerequisites
- You can open **n8n** in your browser.
- A Google Sheet exists (or create one) with **headers** in row 1:
  ```
  date_ist | signups | revenue_inr | note
  ```
- You have a **Google OAuth2** credential configured in n8n.
- You know your **Spreadsheet ID** (from the Sheet URL between `/d/` and `/edit`).

---

## 1) Create the workflow
**1.1** Top bar → **Workflows** → **+ New**  
**1.2** Rename: `Daily KPI Log (09:00 IST)`

---

## 2) Add **Schedule Trigger**
**2.1** Press **A** (or click **+**) → search **schedule** → select **Schedule Trigger**  
**2.2** Configure:
- **Mode** → **Specific times**  
- **Add time** → **Hour = 9**, **Minute = 0**  
- **Timezone** → **Asia/Kolkata**  
**2.3** Rename node → **Schedule (Daily 09:00 IST)**

> 🕘 Fires once per day at **09:00 IST**.

---

## 3) Add **Function** — build the KPI row
**3.1** Add **Function** node to the right of Schedule  
**3.2** Connect → **Schedule → Function**  
**3.3** Rename → **Build Row**  
**3.4** Paste this code:
```js
const now = Date.now();
const istIso = new Date(now + 19800000).toISOString(); // UTC +5:30 (IST)

// Replace these with real metrics later (DB/API/previous nodes)
return [{
  json: {
    date_ist: istIso.slice(0,10), // YYYY-MM-DD
    signups: 12,
    revenue_inr: 34990,
    note: 'Daily KPI log'
  }
}];
```

---

## 4) Add **Google Sheets** — append the row
**4.1** Add **Google Sheets** node to the right of **Build Row**  
**4.2** Connect → **Build Row → Google Sheets**  
**4.3** Configure:
- **Operation** → **Append**  
- **Spreadsheet** (`documentId`) → **your Spreadsheet ID**  
- **Range** → `Sheet1!A:D`  
- **Options → Value Input Mode** → `RAW`  
- **Credentials** → select your **Google OAuth2** credential  
- **Map columns** (if prompted):
  - **A** → `{{ $json.date_ist }}`
  - **B** → `{{ $json.signups }}`
  - **C** → `{{ $json.revenue_inr }}`
  - **D** → `{{ $json.note }}`

---

## 5) Test → Activate
**5.1** Click **Execute Workflow** → confirm a new row appears in your Sheet  
**5.2** Toggle **Activate** (top-right) → runs daily at **09:00 IST**

---

## (Optional) Bulletproof error handling
**Pattern:**  
`Google Sheets (Continue On Fail = ON)` → **IF** (`{{$json.error}}` is **empty?**)  
- **TRUE →** OK path  
- **FALSE →** Recover path (e.g., **Wait** 10s → **retry**, or **send alert**)

---

## Troubleshooting
- **Nothing runs** → workflow is **Inactive**. Toggle **Activate**.  
- **Wrong time** → check **Timezone = Asia/Kolkata**.  
- **Permission error** → pick the correct **Google OAuth2** credential.  
- **Range not found** → confirm `Sheet1` exists and range is `Sheet1!A:D`.  
- **Headers missing** → add `date_ist | signups | revenue_inr | note` to row 1.

---

## 📥 Import-ready workflow JSON
Paste via **Workflows → Import from file/clipboard**.  
After import, open the **Google Sheets** node: select your credential and **replace** `REPLACE_WITH_YOUR_SPREADSHEET_ID`.

```json
{
  "name": "Daily KPI Log (09:00 IST) → Google Sheets",
  "nodes": [
    {
      "parameters": {
        "mode": "specific",
        "triggerTimes": {
          "item": [
            { "hour": 9, "minute": 0 }
          ]
        },
        "timezone": "Asia/Kolkata"
      },
      "name": "Schedule (Daily 09:00 IST)",
      "type": "n8n-nodes-base.cron",
      "typeVersion": 1,
      "position": [200, 280],
      "id": "ScheduleDaily"
    },
    {
      "parameters": {
        "functionCode": "const now = Date.now();\nconst istIso = new Date(now + 19800000).toISOString(); // UTC+5:30 (IST)\n// Replace these demo values with real KPIs later\nreturn [{ json: {\n  date_ist: istIso.slice(0,10),\n  signups: 12,\n  revenue_inr: 34990,\n  note: 'Daily KPI log'\n}}];"
      },
      "name": "Build Row",
      "type": "n8n-nodes-base.function",
      "typeVersion": 2,
      "position": [460, 280],
      "id": "FnRow"
    },
    {
      "parameters": {
        "operation": "append",
        "documentId": "REPLACE_WITH_YOUR_SPREADSHEET_ID",
        "range": "Sheet1!A:D",
        "options": {
          "valueInputMode": "RAW"
        }
      },
      "name": "Google Sheets - Append Row",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 6,
      "position": [720, 280],
      "credentials": {
        "googleSheetsOAuth2Api": {
          "id": "",
          "name": ""
        }
      },
      "id": "AppendRow"
    }
  ],
  "connections": {
    "Schedule (Daily 09:00 IST)": {
      "main": [
        [
          { "node": "Build Row", "type": "main", "index": 0 }
        ]
      ]
    },
    "Build Row": {
      "main": [
        [
          { "node": "Google Sheets - Append Row", "type": "main", "index": 0 }
        ]
      ]
    }
  },
  "active": false,
  "version": 2
}
```
