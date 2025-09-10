# Task 2 — Sheet Connections (n8n Form → Google Sheets)

Audience: First‑time n8n users and non‑technical readers. Steps use clear sub‑steps with `+` and `→`. Drag‑and‑drop requirements are explicitly noted.

---

## Important Note (Before You Start)

```text
+ You must authenticate with Google in n8n (Credentials → Google → Google Sheets).
+ In Google Cloud Console, enable the "Google Sheets API" for your project.
+ During OAuth setup, add the redirect URI:
  https://<your-n8n-host>/rest/oauth2-credential/callback
```

---

## Goal

Collect a **Daily Tech Standup** using an **n8n Form**, then **append each submission as a new row** in a Google Sheet.

**Flow (text mockup)**
```
[Form Trigger] → [Google Sheets: Append Row]
```

---

## STEP 0 — Create the Google Sheet (manual)

### Do
```text
+ Open Google Sheets → New sheet
  → Rename spreadsheet: Daily Standup Log
  → Rename first tab (sheet): Standups
  → Add headers in Row 1 (exactly):
    employee_name | email | date_iso | project | yesterday | today | blockers | priority | tags
```
### Copy its URL
```text
+ From your browser address bar, copy the full spreadsheet URL
  (we will paste it into the Google Sheets node as "By URL")
```

**Example of a filled sheet (illustrative):**
![Sheet result](sandbox:/mnt/data/Screenshot 2025-09-10 at 3.02.46 PM.png)

---

## STEP 1 — Form Trigger (collect inputs)  (drag‑and‑drop: required)

### Do
```text
+ Left panel search: Form Trigger
  → Drag to canvas (leftmost)
  → Click to configure
```

### Configure (fields for a daily tech use case)
```text
+ Path        : forms/standup
+ Form Title  : Daily Tech Standup
+ Description : Quick daily report for the engineering team
+ Fields (Add Field for each):
  → Employee Name   (Text, required)     → Name: employee_name
  → Email           (Email, required)    → Name: email
  → Project         (Text, required)     → Name: project
  → Yesterday       (Textarea, required) → Name: yesterday
  → Today           (Textarea, required) → Name: today
  → Blockers        (Textarea, optional) → Name: blockers
  → Priority        (Select, required)   → Name: priority (options: low, medium, high)
  → Tags            (Text, optional)     → Name: tags (comma separated)
```
**Tip**
```text
+ You can also add a read‑only hidden field "date_iso" using a Set node later (={{ $now }}) — or ask the user to pick a date.
```

**Reference (UI preview)**
![Form + mapping preview](sandbox:/mnt/data/FE8429A5-7111-4E42-9001-B8094A3CA390.png)

---

## STEP 2 — Google Sheets: Append Row  (drag‑and‑drop + connection: required)

### Do
```text
+ Left panel search: Google Sheets
  → Drag to the canvas (right of Form Trigger)
  → Connect: Form Trigger → Google Sheets
```

### Configure (Append Row)
```text
+ Resource            : Sheet Within Document
+ Operation           : Append Row
+ Document            : By URL → paste the spreadsheet URL from STEP 0
+ Sheet               : From list → Standups
+ Mapping Column Mode : Map Each Column Manually
+ Values to Send      :
  → employee_name = ={{ $json["employee_name"] }}
  → email        = ={{ $json["email"] }}
  → date_iso     = ={{ $now }}
  → project      = ={{ $json["project"] }}
  → yesterday    = ={{ $json["yesterday"] }}
  → today        = ={{ $json["today"] }}
  → blockers     = ={{ $json["blockers"] }}
  → priority     = ={{ $json["priority"] }}
  → tags         = ={{ $json["tags"] }}
```

**Illustration of node config and output:**
![Append node config](sandbox:/mnt/data/A21A92A7-411B-42A1-A17C-03E5D12B9B42.png)
![Canvas view](sandbox:/mnt/data/B43B5E0B-C3FB-4721-A690-71D94A444225.png)

---

## STEP 3 — Run and Verify

```text
+ Click Save
+ Click Execute workflow (top‑right)
+ Click “Open form” on Form Trigger → fill and submit
+ Verify a new row appears in your Google Sheet (tab: Standups)
```

---

## Troubleshooting

```text
+ "No permission" or auth popup fails:
  → Recreate the Google credential and ensure Sheets API is enabled.
+ Sheet not found:
  → Make sure you used "By URL" and the account has access to the file.
+ Mapped field empty in the row:
  → Check the exact form "Name" matches the key used in expressions.
+ Wrong tab:
  → Ensure the selected Sheet is "Standups" (matches your tab name).
```

---

## Starter Workflow JSON (import‑ready)

> Import via **n8n → Import → Paste JSON**. After import, open the **Google Sheets** node and select/finish your Google credential. If your n8n version shows slightly different Google node fields, re‑select the same operation and remap columns in the UI.

```json
{
  "name": "Daily Standup to Google Sheets",
  "nodes": [
    {
      "parameters": {
        "path": "forms/standup",
        "formTitle": "Daily Tech Standup",
        "formDescription": "Quick daily report for the engineering team",
        "formFields": {
          "values": [
            { "fieldLabel": "Employee Name", "name": "employee_name", "placeholder": "Your full name", "requiredField": true },
            { "fieldLabel": "Email", "name": "email", "fieldType": "email", "placeholder": "you@company.com", "requiredField": true },
            { "fieldLabel": "Project", "name": "project", "placeholder": "Project/Service name", "requiredField": true },
            { "fieldLabel": "Yesterday", "name": "yesterday", "fieldType": "textarea", "placeholder": "What did you complete yesterday?", "requiredField": true },
            { "fieldLabel": "Today", "name": "today", "fieldType": "textarea", "placeholder": "What will you do today?", "requiredField": true },
            { "fieldLabel": "Blockers", "name": "blockers", "fieldType": "textarea", "placeholder": "Anything blocking you?", "requiredField": false },
            { "fieldLabel": "Priority", "name": "priority", "fieldType": "select", "options": { "options": [ { "name": "low" }, { "name": "medium" }, { "name": "high" } ] }, "requiredField": true },
            { "fieldLabel": "Tags", "name": "tags", "placeholder": "comma,separated,tags", "requiredField": false }
          ]
        },
        "options": {}
      },
      "id": "formStandup",
      "name": "Form Trigger",
      "type": "n8n-nodes-base.formTrigger",
      "typeVersion": 1,
      "position": [ -140, -160 ]
    },
    {
      "parameters": {
        "operation": "append",
        "sheetId": "",
        "range": "",
        "options": {
          "locationDefine": "define",
          "valueInputMode": "RAW",
          "keyRow": 1
        },
        "columns": {
          "string": [
            { "name": "employee_name", "value": "={{ $json[\"employee_name\"] }}" },
            { "name": "email",          "value": "={{ $json[\"email\"] }}" },
            { "name": "date_iso",       "value": "={{ $now }}" },
            { "name": "project",        "value": "={{ $json[\"project\"] }}" },
            { "name": "yesterday",      "value": "={{ $json[\"yesterday\"] }}" },
            { "name": "today",          "value": "={{ $json[\"today\"] }}" },
            { "name": "blockers",       "value": "={{ $json[\"blockers\"] }}" },
            { "name": "priority",       "value": "={{ $json[\"priority\"] }}" },
            { "name": "tags",           "value": "={{ $json[\"tags\"] }}" }
          ]
        }
      },
      "id": "gsheetsAppend",
      "name": "Google Sheets (Append Row)",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [ 220, -160 ],
      "credentials": {
        "googleApi": {
          "id": "__PLEASE_CONFIGURE__",
          "name": "Google Account"
        }
      }
    }
  ],
  "connections": {
    "Form Trigger": {
      "main": [
        [
          { "node": "Google Sheets (Append Row)", "type": "main", "index": 0 }
        ]
      ]
    }
  },
  "active": false,
  "settings": {}
}
```
