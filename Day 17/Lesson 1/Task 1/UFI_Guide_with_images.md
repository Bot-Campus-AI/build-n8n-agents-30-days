# Universal Form Intelligence (n8n) — Reminder / Database / Broadcast

**One-line goal:** turn a single form into three actions—create a Calendar event, log to MySQL, or broadcast to Slack—with LLM assist.

---

## At-a-glance outcomes
- Single **Form Trigger** drives three paths via **Switch** (`Reminder | Database | Broadcast`).
- **Reminder** → LLM converts natural text → RFC3339 → Google Calendar event.
- **Database** → Append name/email/data to **MySQL** with large text support.
- **Broadcast** → LLM rewrites message in a professional tone → Slack channel post.

---

## Prereqs & Auth (do this first)
➜ a. **Slack setup video:** https://www.botcampus.ai/lessons/slack-authentication-and-workflow  
➜ b. **MySQL install video:** https://www.botcampus.ai/lessons/mysql-authentication-and-workflow

**Also required**
- **n8n** (Cloud or self-hosted).
- **Google Calendar OAuth2** in n8n  
  - Enable **Google Calendar API** in Google Cloud.  
  - Create OAuth Client (Web).  
  - In n8n **Credentials → Google Calendar OAuth2**, grant scope: `https://www.googleapis.com/auth/calendar.events`.
- **Slack** credential in n8n  
  - Slack App → add scope `chat:write`, install to workspace → paste Bot token in n8n Slack credential (or OAuth2).
- **MySQL 8.x** reachable from n8n  
  - Create DB (e.g., `n8n`), user with `SELECT/INSERT/ALTER`.
- **Gemini API key** in n8n  
  - **Credentials → Google PaLM (Gemini)**; use a fast model (e.g., *Gemini 2.5-flash*).
- **Timezone default:** Asia/Kolkata (IST) used by Reminder path.

---

## Architecture snapshot (nodes/tools)
1) **On form submission (Form Trigger)** → fields: **Name**, **Action (Dropdown)**, **Event description**, **Email**  
2) **Switch** on `Action`  
   - **Reminder** → *Basic LLM Chain (Gemini)* → **Code (normalize JSON)** → **Google Calendar: Create an event**  
   - **Broadcast** → *Basic LLM Chain1 (Gemini)* → **Slack: Send a message**  
   - **Database** → **MySQL: Insert rows in a table**

---

## Your run screenshots (reference)
![Calendar result](ufi_images/calendar.png)  
*Figure 1 — Event created 10–11 PM IST.*

![Workflow canvas](ufi_images/canvas.png)  
*Figure 2 — Final n8n canvas with three branches.*

![MySQL rows](ufi_images/mysql.png)  
*Figure 3 — Rows inserted into `slack` table.*

![Slack post](ufi_images/slack.png)  
*Figure 4 — Broadcast message posted to `#n8n-alerts`.*

---

## Step-by-Step (with in-place copy blocks)

### 1) Form Trigger — clean field labels & options
- **Form fields (exact labels):** `Name`, `Action`, `Event description`, `Email`  
- **Action (Dropdown) options (case sensitive):** `Reminder`, `Database`, `Broadcast`  
- If you already created **`Event description `** *(note trailing space)*, either rename it to **`Event description`** or keep as-is; in later expressions we’ll read both keys.

**Where to paste (Form Trigger → Fields)**  
_No code—verify labels and dropdown values exactly as above._

---

### 2) Switch — fix comparisons (remove stray `=`)
Your JSON had `=Reminder` and `=Database` in `rightValue`. Remove the `=`. Keep **Case Sensitive = true**.

**Set these rules (Switch → Rules):**
```
Left:  {{$json.Action}}   Operator: equals   Right: Reminder
Left:  {{$json.Action}}   Operator: equals   Right: Broadcast
Left:  {{$json.Action}}   Operator: equals   Right: Database
```

*(Optional) Add a **Default** output → small Respond/Slack message: “Pick a valid Action.”*

---

### 3) Reminder path — LLM → normalized JSON → Calendar

**3a) Basic LLM Chain (Gemini) → `text` field**  
_Paste this prompt (keeps output one-line JSON and IST logic):_
```
You convert ONE natural-language reminder into ONE Google Calendar–ready payload.

OUTPUT: RETURN EXACTLY 1 JSON OBJECT ON ONE LINE. NO MARKDOWN. NO FENCES. NO EXTRA TEXT.
SCHEMA (exact keys):
{
  "startDateTime": "YYYY-MM-DDTHH:mm:ss.SSS±HH:mm",
  "endDateTime":   "YYYY-MM-DDTHH:mm:ss.SSS±HH:mm",
  "summary":       "string",
  "description":   "string"
}

RULES:
- Timezone default: Asia/Kolkata (+05:30) unless another TZ/city is explicitly given.
- If only a time is given (no date) → use TODAY in Asia/Kolkata.
- Relative words:
  • today/tonight/this evening/morning/afternoon → today (IST)
  • tomorrow → tomorrow (IST)
  • next <weekday> → next occurrence after NOW_IST (IST)
- Durations:
  • If “for N minutes/hours” → end = start + duration
  • Else default duration = 60 minutes
- Summary: short, capitalized; remove date/time words.
- Description: concise remaining context; no date/time strings.

INPUT_TEXT: "{{ $json['Event description'] || $json['Event description '] }}"
NOW_IST: "{{ $now.setZone('Asia/Kolkata').toISO() }}"
Return only the JSON object.
```

**Model connection (dotted line):** Connect **Google Gemini Chat Model** to this Chain via **AI Model** input.

**3b) Code (JavaScript) — normalize AI output**  
_Paste this into the **Code** node:_
```javascript
function coalesce(...v){ for (const x of v) if (x!==undefined && x!==null && String(x).trim()!=='') return x }
function stripFences(s){ return String(s).replace(/^\s*```[a-zA-Z]*\s*|\s*```\s*$/g,'').trim() }
function tryJson(s){
  s = stripFences(String(s));
  try { return JSON.parse(s) } catch {}
  const m = s.match(/\{[\s\S]*\}/); if (m) { try { return JSON.parse(m[0]) } catch {} }
  return null
}
function bracketDT(s){ const m = String(s).match(/\[DateTime:\s*([^\]]+)\]/i); return m ? m[1] : null }
function pad(n){ return String(n).padStart(2,'0') }
function ms3(n){ return String(n).padStart(3,'0') }
function addMinutesKeepOffset(iso, mins){
  const off = (String(iso).match(/([+-]\d{2}:\d{2})$/) || [,'+05:30'])[1];
  const sign = off.startsWith('-') ? -1 : 1;
  const [H,M] = off.slice(1).split(':').map(Number);
  const offMin = sign*(H*60+M);
  const start = new Date(iso);
  const endUtc = new Date(start.getTime() + mins*60*1000);
  const endLocal = new Date(endUtc.getTime() + offMin*60*1000);
  const yyyy = endLocal.getUTCFullYear(), MM = pad(endLocal.getUTCMonth()+1), dd = pad(endLocal.getUTCDate());
  const HH = pad(endLocal.getUTCHours()), mm = pad(endLocal.getUTCMinutes()), ss = pad(endLocal.getUTCSeconds()), SSS = ms3(endLocal.getUTCMilliseconds());
  return `${yyyy}-${MM}-${dd}T${HH}:${mm}:${ss}.${SSS}${off}`;
}
function titleCase(s){ return String(s||'').trim().replace(/\s+/g,' ').replace(/\w/g,c=>c.toUpperCase()) }

const out = [];
for (const [i, item] of items.entries()){
  const input = item.json || {};
  const originalText = coalesce(input['Event description'], input['Event description '], input.event_description, input.prompt, input.query, '');
  const rawAi = coalesce(
    input.ai, input.answer, input.text, input.output, input.response, input.message,
    input.content, input.result, input.data, input.choices?.[0]?.message?.content,
    typeof input === 'string' ? input : JSON.stringify(input)
  );

  let p = (typeof rawAi === 'object') ? rawAi : tryJson(rawAi);
  let startDateTime, endDateTime, summary, description;

  if (p && typeof p === 'object'){
    ({ startDateTime, endDateTime, summary, description } = p);
  } else {
    const b = bracketDT(rawAi);
    if (!b) throw new Error('AI output not JSON and no [DateTime: …] found.');
    startDateTime = b;
  }

  if (!startDateTime || isNaN(new Date(startDateTime).getTime())) throw new Error('Invalid startDateTime');

  if (!summary) summary = titleCase((originalText||'').split(/[.!?]/)[0] || 'Reminder');
  if (!description) description = String(originalText||'').trim();

  if (!endDateTime) endDateTime = addMinutesKeepOffset(startDateTime, 60);
  else if (isNaN(new Date(endDateTime).getTime())) throw new Error('Invalid endDateTime');

  out.push({ json: { startDateTime, endDateTime, summary, description }, pairedItem: { item: i }});
}
return out;
```

**3c) Google Calendar — Create an event**
```
Calendar     = (your calendar id)
Start        = {{ $json.startDateTime }}
End          = {{ $json.endDateTime }}
Summary      = {{ $json.summary }}
Description  = {{ $json.description }}
```

---

### 4) Broadcast path — LLM rewrite → Slack

**4a) Basic LLM Chain1 (Gemini) → `text` field**  
_Paste the rewrite instruction:_
```
Rewrite the user’s short note into a professional, medium-length Slack update (80–150 words).
Tone: formal yet approachable. Focus on purpose, context, stakeholders, next steps.

Input:
"{{ $json['Event description'] || $json['Event description '] }}"

Guidelines:
- Expand acronyms if unclear.
- Clarify missing context (what/why/who/when) if implied by the text.
- Avoid fluff; keep it factual and helpful.
- End with an explicit call-to-action or next step when relevant.

Return only the rewritten paragraph as plain text (no markdown).
```

**Model connection:** Connect **Google Gemini Chat Model1** to this Chain via **AI Model** input.

**4b) Slack — Send a message**
```
Channel: (your channel or ID, e.g., C09HNRB0KMH)
Text:    {{ $json.text || $json.response || $json.output }}
```

---

### 5) Database path — validate email → insert MySQL
*(Optional but recommended) Add an IF node before insert: if `{{$json.Email}}` is empty AND `{{$json.Action}} == 'Database'` → route to a Slack/Respond node: “Email required for Database action.”*

**5a) MySQL — Insert rows in a table**
```
Table: slack

name        = {{ $json.Name }}
email       = {{ $json.Email || '' }}
data        = {{ $json['Event description'] || $json['Event description '] || '' }}
created_at  = {{ $json.submittedAt || $now.toISO() }}
```

**5b) DDL for the table (run once in MySQL Workbench)**
```sql
CREATE TABLE IF NOT EXISTS slack (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(320) NULL,
  data LONGTEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

*(If you previously had a UNIQUE constraint on email, drop it and widen `data`):*
```sql
-- Drop unique index if present (use the actual index name from SHOW INDEX)
ALTER TABLE slack DROP INDEX email;           -- or DROP INDEX email_unique;
-- Widen data
ALTER TABLE slack MODIFY COLUMN data LONGTEXT NOT NULL;
-- Create non-unique helper index
CREATE INDEX idx_email ON slack (email);
```

---

## Node list (order) & key parameters

| # | Node | Key settings / expressions |
|---|------|----------------------------|
| 1 | On form submission (Form Trigger) | Fields: Name, Action=Dropdown(Reminder,Database,Broadcast), Event description, Email |
| 2 | Switch | Three rules: `{{$json.Action}} == Reminder`, `== Broadcast`, `== Database` |
| 3 | Google Gemini Chat Model | Credential: *Gemini 2.5-flash* |
| 4 | Basic LLM Chain (Reminder) | Prompt from Step 3a |
| 5 | Code (JavaScript) | Normalizer code from Step 3b |
| 6 | Google Calendar: Create an event | Start/End/Summary/Description from JSON |
| 7 | Google Gemini Chat Model1 | Credential: *Gemini 2.5-flash* |
| 8 | Basic LLM Chain1 (Broadcast) | Prompt from Step 4a |
| 9 | Slack: Send a message | Text: `{{$json.text || $json.response || $json.output}}` |
| 10 | MySQL: Insert rows in a table | Mapping from Step 5a; credential “MySQL account” |

---

## Testing & Validation

**A) Reminder**  
- Action: `Reminder`  
- Event description: `Dinner with a friend at 10:00 PM tonight near Jubilee Hills`  
- Expect: Calendar event today **22:00–23:00 IST**. *(See Figure 1)*

**B) Database**  
- Action: `Database`  
- Name: `Jashwanth`  
- Email: `jashwanthboddupally@gmail.com`  
- Event description: _any long text_  
- Expect: New row in `slack`. *(See Figure 3)*

**C) Broadcast**  
- Action: `Broadcast`  
- Event description: `Standup moved to 10:30 AM; infra deploy at 3 PM. QA to retest build 142.`  
- Expect: Professional message in Slack. *(See Figure 4)*

---

## Troubleshooting

- Switch not routing → ensure right values are exactly `Reminder`, `Database`, `Broadcast` (no leading `=`).  
- LLM returns markdown → Code node strips fences and extracts JSON; still ensure the Chain prompt forbids markdown.  
- Calendar “Invalid DateTime” → Code node must run *before* Calendar node; verify JSON keys.  
- Empty Slack message → `{{$json.text || $json.response || $json.output}}`.  
- MySQL insert fails → run the DDL, confirm table and credential, ensure `data` isn’t NULL.  
- Form field mismatch → if you kept `Event description ` (with space), always reference both keys as shown.

---

## What to Deliver
- Fixed **Switch** comparisons, clean **Form** fields.  
- **Reminder**: Gemini Chain → Code normalizer → Calendar event.  
- **Broadcast**: Gemini Chain1 → Slack message.  
- **Database**: MySQL DDL + Insert mapping.  
- Screens consistent with **Figures 1–4**.
