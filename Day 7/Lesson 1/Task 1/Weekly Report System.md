# 📊 Weekly Report Inbox → Executive Report System (n8n)

Automates weekly team updates into a polished executive report.

---

## ⚙️ Architecture
Cron (Friday 17:00 IST) ➝ Airtable (Updates) ➝ OpenAI Chat ➝ Google Docs (Create + Append) ➝ Gmail Send

---

## 🧩 Setup Instructions

### ⏰ Step 1 — Schedule Trigger
- Drag Cron Node
- Mode: Every Week
- Day: Friday
- Time: 17:00
- Timezone: Asia/Kolkata
- Rename: `Weekly Trigger (Fri 5PM)`

---

### 📥 Step 2 — Airtable (Collect Updates)
- Drag Airtable Node
- Operation: List Records
- Base: `Weekly Report Inbox`
- Table: `Updates`
- Filter Formula:
  ```
  IS_AFTER({Created}, DATEADD(TODAY(), -7, 'days'))
  ```
- Output: JSON array (last 7 days of entries)
- Rename: `Fetch Weekly Updates`

**Airtable Schema (table: Updates):**
- Team (Single Select)
- Update (Long Text)
- KPIs (Long Text, optional)
- Links (URL, optional)
- Created (Created Time)

---

### 🧠 Step 3 — OpenAI Chat (Summarize)
- Drag OpenAI Chat Node (or Anthropic/Gemini)

**System Prompt:**
```
You turn weekly raw updates into a crisp executive report.
Sections: 1) Highlights 2) KPIs 3) Risks/Blockers 4) Next Week Focus 5) Team Shoutouts.
Rules: be concise, group similar items, convert bullets to clear prose/bullets,
normalize numbers, remove duplicates, keep links.
Output in clean Markdown. No preamble.
```

**User Prompt:**
```
Here are this week's submissions as JSON:
{{$json}}
```

- Rename: `LLM — Executive Report`

---

### 📄 Step 4 — Google Docs (Create + Append)
#### 4.1 Create
- Google Docs Node → Create Document
- Title: `Weekly Report — {{$now.format("YYYY-MM-DD")}}`
- Rename: `Create Report Doc`

#### 4.2 Append
- Google Docs Node → Append Document
- Document ID: `={{$node["Create Report Doc"].json.documentId}}`
- Content:
  ```
  {{$json["choices"][0]["message"]["content"] || $json.data || $json}}
  ```
- Rename: `Append Report Content`

---

### 📧 Step 5 — Gmail (Distribute)
- Gmail Node → Send Email
- To:
  ```
  stakeholder1@company.com, stakeholder2@company.com
  ```
- Subject:
  ```
  Weekly Report — {{$now.format("MMM Do, YYYY")}}
  ```
- Body (HTML):
  ```html
  Hi all,<br><br>
  Here’s the weekly report: 
  <a href="{{$node["Create Report Doc"].json.webViewLink}}">Open in Google Docs</a><br><br>
  — BotCampus AI Weekly Automations
  ```
- Rename: `Email Stakeholders`

---

## 🚀 Optional Add-ons
- Slack Node: Post “Report Ready” with Doc link
- Google Drive → Export File (PDF): attach PDF
- IF Node (Fallback): If Airtable returns 0 items → send “No updates this week.”

---

✅ **Final Flow:**  
Friday 5 PM → Cron fires → Airtable updates pulled → LLM summarizes → Report saved in Google Doc → Email sent to stakeholders
