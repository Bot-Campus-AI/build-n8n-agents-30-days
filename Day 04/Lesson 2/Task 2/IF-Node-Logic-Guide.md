#  IF Node Logic with Urgent Keyword — Demo Workflow

# IF–Node Email Triage (Urgent + Domain) — n8n
**Goal:** Route email-like data using IF logic → (Urgent+Company) alert, (Company) priority, else standard.

![Editor overview – expected layout](images/01-if-logic-demo.png)

---

## At-a-glance outcomes
- Detect **“urgent”** in subject (case-insensitive).
- Extract **sender domain** and **sender name**.
- Two IF gates → **3 outcomes** (Alert / Priority / Standard).
- Send Gmail notifications/replies.
- Import-ready workflow JSON included.

---

## Prereqs & Auth (do this first)
- **n8n:** Cloud or Self-hosted (recent version).
- **Gmail OAuth2 credential**
  1) In n8n go to **Credentials → New → Gmail OAuth2 API → Connect** (n8n Cloud).
  2) **Self-hosted only:** In Google Cloud, enable **Gmail API** → create **OAuth Client (Web)** → add redirect:
     ```
     https://YOUR_N8N_DOMAIN/rest/oauth2-credential/callback
     ```
     Paste **Client ID** and **Client Secret** in n8n → **Connect**.
  - Name the credential (e.g., **Gmail – Personal**). Use this in Gmail nodes.

---

## Architecture snapshot (nodes/tools)
**Order & flow**
1) **Manual Trigger**  
2) **Set – Email Data** → sample sender/subject/body  
3) **Set – Email Data1** → preserve raw fields for Gmail  
4) **Set – Extract & Clean** → derive flags/parts  
5) **IF – Urgent + Company?** (AND)  
   - **True** → 6) **Set – Urgent Notification1** → 7) **Gmail – Send a message (alert to you)**  
   - **False** → 8) **IF – Company?**  
     - **True** → 9) **Set – Priority Inbox1** → 10) **Gmail – Send a message1 (reply to sender)**  
     - **False** → 11) **Set – Standard Queue** → 12) **NoOp**

**Branching behavior**
- Urgent+Company → **Alert path**  
- Company only → **Priority path**  
- Otherwise → **Standard path**

---

## Step-by-Step

### 1) Manual Trigger
➜ a. Drag **Manual Trigger**.  
➜ b. Rename: **When clicking ‘Execute workflow’**.

---

### 2) Seed data — **Set: Email Data**
➜ a. Add **Set** → **Add Fields → String**: `sender`, `subject`, `body`.  
➜ b. Paste this sample (adjust anytime):

```text
sender = jashu@botcampus.com
subject = URGENT: Q4 Report Needed
body =
Hi Abdullah,

I’m going through Module 4 and had a doubt regarding the IF node conditions and decision-making tasks. Could you please clarify how the conditions are applied and how to structure them properly in workflows?

Thanks in advance for your guidance.
```

➜ c. **Keep Only Set**: Off (default).  
➜ d. Node name: **Email Data**.

---

### 3) Preserve for Gmail — **Set: Email Data1**
➜ a. Add **Set**.  
➜ b. **Keep Only Set**: On.  
➜ c. Add these **String** fields (exact expressions):

```text
sender  = {{ $json.sender }}
subject = {{ $json.subject }}
body    = {{ $json.body }}
```

➜ d. Connect **Email Data → Email Data1**.

---

### 4) Derive fields — **Set: Extract & Clean**
➜ a. Add **Set**.  
➜ b. **Keep Only Set**: On.  
➜ c. Add fields exactly as below (copy-paste):

```text
sender_domain = {{ ($json["sender"] || "").split("@")[1] || "" }}
subject_clean = {{ ($json["subject"] || "").toLowerCase() }}
sender_name   = {{ ($json["sender"] || "").split("@")[0] || "" }}
is_urgent     = {{ ($json["subject"] || "").toLowerCase().includes("urgent") }}
```

➜ d. Connect **Email Data1 → Extract & Clean**.

---

### 5) First gate — **IF: Urgent + Company?** (AND)
➜ a. Add **IF**; name it **Urgent + Company?**  
➜ b. Add **two** conditions:

```text
[Boolean → Equals]
Left  = {{ $json.is_urgent }}
Right = true
```
```text
[String → Equals]
Left  = {{ $json.sender_domain }}
Right = botcampus.com        ← use your company domain
```

➜ c. Connect **Extract & Clean → Urgent + Company?**.

---

### 6) True branch payload — **Set: Urgent Notification1**
➜ a. **Keep Only Set**: On.  
➜ b. Add fields:

```text
category = Urgent {{ $json.is_urgent }}
action   = immediate_notification
message  = 🚨 URGENT: Email {{ $json.subject_clean }} from  {{ $json.sender_name }}
```

---

### 7) True branch email — **Gmail: Send a message** (alert to you)
Use your Gmail credential (e.g., **Gmail – Personal**). Set:

```text
To         = your@email.com
Subject    = {{ $json.message }}
Email Type = Text
Message    = {{ $('Email Data1').item.json.body }}
```

Wire: **Urgent + Company? (true) → Urgent Notification1 → Send a message**.

---

### 8) False branch second gate — **IF: Company?**
➜ a. Add **IF**; name **Company?**  
➜ b. Single condition (same domain as Step 5):

```text
[String → Equals]
Left  = {{ $json.sender_domain }}
Right = botcampus.com
```

Wire: **Urgent + Company? (false) → Company?**.

---

### 9) Company true payload — **Set: Priority Inbox1**
➜ a. **Keep Only Set**: On.  
➜ b. Fields:

```text
category = company_priority
action   = add_to_priority_inbox
message  = {{ "⚠️ Company email from " + $json["sender_name"] + " - " + $json["subject"] }}
```

---

### 10) Company true email — **Gmail: Send a message1** (reply to sender)
Use the same Gmail credential. Set:

```text
To         = {{ $('Email Data1').item.json.sender }}
Subject    = {{ $('Email Data1').item.json.subject }}
Email Type = Text
Message    = {{ $('Email Data1').item.json.body }}
```

Wire: **Company? (true) → Priority Inbox1 → Send a message1**.

---

### 11) Company false payload — **Set: Standard Queue**
➜ a. **Keep Only Set**: On.  
➜ b. Fields:

```text
category = external
action   = standard_processing
message  = {{ "📧 External email from " + $json["sender_name"] + " - " + $json["subject"] }}
```

---

### 12) Sink — **No Operation, do nothing**
Wire: **Standard Queue → NoOp**.

---

## Copy-Paste Reference (all snippets together)
**Expressions**
```text
sender_domain = {{ ($json["sender"] || "").split("@")[1] || "" }}
subject_clean = {{ ($json["subject"] || "").toLowerCase() }}
sender_name   = {{ ($json["sender"] || "").split("@")[0] || "" }}
is_urgent     = {{ ($json["subject"] || "").toLowerCase().includes("urgent") }}
```

**IF conditions**
```text
Urgent + Company?:
  {{ $json.is_urgent }} == true
  {{ $json.sender_domain }} == botcampus.com
Company?:
  {{ $json.sender_domain }} == botcampus.com
```

**Gmail bindings**
```text
Send a message   → Subject = {{ $json.message }}
Send a message   → Message = {{ $('Email Data1').item.json.body }}
Send a message1  → To      = {{ $('Email Data1').item.json.sender }}
Send a message1  → Subject = {{ $('Email Data1').item.json.subject }}
Send a message1  → Message = {{ $('Email Data1').item.json.body }}
```

---

## Testing & Validation
1) **Urgent + Company** → Subject `URGENT: ...`, Sender `name@botcampus.com` → expect **alert email**.  
2) **Company only** → Subject `FYI: ...`, Sender `name@botcampus.com` → expect **reply path**.  
3) **External** → Sender `name@gmail.com` → expect **Standard Queue → NoOp**.  
4) **Edge** → Empty subject → `is_urgent=false`; malformed sender → `sender_domain=""` → external.

---

## Troubleshooting
- **401/403 Gmail** → Reconnect credential; ensure the same Google account granted access.  
- **Redirect mismatch** (self-hosted) → add exact callback URL shown above.  
- **No match on company** → both IF nodes must use **the same domain**.  
- **Undefined property** → keep the provided null-safe expressions; **Email Data1** must run before Gmail nodes.  
- **No email received** → check Spam; ensure Gmail node is in **Send** mode and **Email Type = Text**.

---

## What to Deliver
- Working triage workflow with 3 outcomes.  
- Gmail OAuth2 credential connected and tested.  
- Company domain consistent in **both** IF nodes.  
- (Optional) Wire each branch to storage (DB/Sheets) for logging.

## Diagram / Screenshot
![IF Logic with Urgent keyword Demo](images/01-if-logic-demo.png)

