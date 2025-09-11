#  IF Node Logic with Urgent Keyword — Demo Workflow

This workflow demonstrates how to classify and act on incoming emails using **IF nodes** in n8n.  
We’ll build a decision tree that routes **urgent company emails**, **regular company emails**, and **external emails** to different actions.

---

##  Workflow Steps

### 1 ➝ Manual Trigger
- Drag & drop **Manual Trigger** node.  
- Purpose: start the workflow whenever you run it manually.

---

### 2 ➝ Set Node: Email Data
- Drag & drop **Set** node → rename to **Email Data**.  
- Connect ➝ `Manual Trigger ➝ Email Data`.  
- **Keep Only Set** = ON.  
- Configure fields:
  - `sender` → `boss@company.com`  
  - `subject` → `URGENT: Q4 Report Needed`  
  - `body` → `Please send the quarterly report by EOD`  
  - `received_time` → `={{ $now }}`  

---

### 3 ➝ Set Node: Email Data1
- Add another **Set** node → rename to **Email Data1**.  
- Connect ➝ `Email Data ➝ Email Data1`.  
- Purpose: ensure email data is passed safely for the next steps.

---

### 4 ➝ Set Node: Extract & Clean
- Drag & drop **Set** node → rename to **Extract & Clean**.  
- Connect ➝ `Email Data1 ➝ Extract & Clean`.  
- **Keep Only Set** = ON.  
- Add fields with **expressions**:

| Field           | Expression |
|-----------------|------------|
| `sender_domain` | `={{ ($json["sender"] || "").split("@")[1] || "" }}` |
| `subject_clean` | `={{ ($json["subject"] || "").toLowerCase() }}` |
| `is_urgent` (Boolean) | `={{ ($json["subject"] || "").toLowerCase().includes("urgent") }}` |
| `sender_name`   | `={{ ($json["sender"] || "").split("@")[0] || "" }}` |

---

### 5 ➝ IF Node: Urgent + Company?
- Drag & drop **IF** node → rename to **Urgent + Company?**.  
- Connect ➝ `Extract & Clean ➝ Urgent + Company?`.  
- Configure conditions (use **AND** logic):
  - Boolean: `={{ $json.is_urgent }} == true`  
  - String: `={{ $json.sender_domain }} == "company.com"`  

**True path** ➝ urgent company emails.  
**False path** ➝ everything else.

---

### 6 ➝ Set Node: Urgent Notification
- For **true branch** of “Urgent + Company?” add **Set** node → rename to **Urgent Notification**.  
- Fields:
  - `category` = `urgent`  
  - `action` = `immediate_notification`  
  - `message` =  
    ```js
    ={{ "🚨 URGENT: Email from " + ($json.sender_name || "unknown") + " - " + ($json.subject || "(no subject)") }}
    ```

---

### 7 ➝ Gmail Node: Send Urgent Message
- Add **Gmail** node → rename to **Send a message**.  
- Connect ➝ `Urgent Notification ➝ Send a message`.  
- Configure:
  - **To:** `team@company.com`  
  - **Subject:** `🚨 URGENT: {{$json.subject}}`  
  - **Body:** short notification with `$json.message`.

---

### 8 ➝ IF Node: Company?
- For **false branch** of “Urgent + Company?” add **IF** node → rename to **Company?**.  
- Condition:  
  - String: `={{ $json.sender_domain }} == "company.com"`

---

### 9 ➝ Priority Inbox (Company non-urgent)
- For **true branch** of “Company?” add **Set** node → rename to **Priority Inbox**.  
- Fields:
  - `category` = `company_priority`  
  - `action` = `add_to_priority_inbox`  
  - `message` =  
    ```js
    ={{ "⚠️ Company email from " + $json.sender_name + " - " + $json.subject }}
    ```  
- Connect to **Gmail node → Send a message1** to notify team.

---

### 10 ➝ Standard Queue (External email)
- For **false branch** of “Company?” add **Set** node → rename to **Standard Queue**.  
- Fields:
  - `category` = `external`  
  - `action` = `standard_processing`  
  - `message` =  
    ```js
    ={{ "📧 External email from " + $json.sender_name + " - " + $json.subject }}
    ```  
- Connect to **No Operation** node → do nothing.

---

##  Workflow Canvas
![Email Output](images/email-output.png)


---

##  Example Email Output
The Gmail node sends structured mail like this:

![Workflow Graph](images/workflow-graph.png)

---

##  Decision Paths
- 🚨 **Urgent + Company** → Immediate notification email  
- ⚠️ **Company (not urgent)** → Priority inbox  
- 📧 **External** → Standard queue (no action)
