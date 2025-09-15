# 📊 Weekly Report Inbox → Executive Report System (n8n)

Automates weekly team updates into a polished executive report.

---

## ⚙️ Architecture
**Cron (Friday 17:00 IST)** ➝ **Airtable (Updates)** ➝ **OpenAI Chat** ➝ **Google Docs (Create + Append)** ➝ **Gmail Send**

---

## 🧩 Step-by-Step Setup

### ⏰ 1. Schedule Trigger
➝ **Drag & Drop ➝ Cron Node**  
- **Mode:** Every Week  
- **Day:** Friday  
- **Time:** 17:00  
- **Timezone:** Asia/Kolkata  
- **Rename:** `Weekly Trigger (Fri 5PM)`

---

### 📥 2. Airtable — Collect Updates
➝ **Drag & Drop ➝ Airtable Node**  
- **Operation:** `List Records`  
- **Base:** `Weekly Report Inbox`  
- **Table:** `Updates`  
- **Filter Formula:**  
