# Weekly Report System — Friday 5:00 PM IST → Drive → LLM → Email (n8n)

**Flow (master rules):**  
**Schedule Trigger (weekly, user-set)** ➝ **Collect (Google Drive)** ➝ **Analyze (LLM of your choice)** ➝ **Generate (formatted report)** ➝ **Distribute (email)**

> 🧭 **Real-world example used below:** “Weekly e-commerce performance”  
> Source files: CSVs in a Google Drive folder (e.g., `Weekly_Reports/Sales_Week_*.csv`)  
> Output: Executive summary + KPIs emailed to stakeholders every **Friday 17:00 IST**.  
> ✏️ You can change both **schedule** and **LLM provider** to your preferences.

---

## 0) One-time prerequisites
- ✅ n8n is running and you can open it in your browser.
- ✅ You have **Google OAuth2** credentials in n8n for **Google Drive** and **Gmail** (or use Email Send/SMTP).
- ✅ Your Google Drive has a folder (e.g., **Weekly_Reports**) containing weekly **CSV** files (columns: `date, order_id, product, category, units, revenue_inr, customer_id`).
- ✅ You have an **LLM credential** in n8n for your preferred model (choose one):
  - **OpenAI** (e.g., `gpt-4o-mini`, `gpt-4o`)  
  - **Anthropic** (Claude)  
  - **Google** (Gemini)  
  - or **HTTP Request** to any compatible API you prefer.

---

## 1) Schedule Trigger (weekly — set to your preference)
**Add ➝ search “schedule” ➝ _Schedule Trigger_**

**Set it (example):**
- **Mode** → **Specific times**
- **Add time** → **Day = Friday**, **Hour = 17**, **Minute = 0**
- **Timezone** → **Asia/Kolkata**
- **Rename**: `Schedule (Weekly Fri 17:00 IST)`

> 💡 You can set a different day/time to match your preference (e.g., Saturday 10:00 IST).

---

## 2) Collect — Google Drive (get the latest weekly CSV)
**Goal:** Find the newest CSV **in a specific folder** and download it.

### 2.1 Find folder (if you don’t know the ID)
- **Node:** Google Drive → **List**  
- **Operation:** *List*  
- **Search** → `Weekly_Reports` (or your folder name)  
- **Tip:** Run once to see the folder `id` in the output, then copy it.

### 2.2 List files in that folder
- **Node:** Google Drive → **List**  
- **List Operation:** *Files*  
- **Folder ID:** *(paste your folder id from 2.1)*  
- **Filter by type:** `mimeType = text/csv` *(or leave default and filter later)*  
- **Include:** `modifiedTime` / `createdTime`  
- **Rename:** `Drive – List Files`

### 2.3 Sort to pick newest
- **Node:** *Item Lists* → **Sort**  
- **Field to sort by:** `modifiedTime` (or `createdTime`)  
- **Order:** *Descending*  
- **Rename:** `Sort by Newest`

### 2.4 Take the first item (the newest file)
- **Node:** *Item Lists* → **Limit**  
- **Max items:** `1`  
- **Rename:** `Pick Latest`

### 2.5 Download the CSV
- **Node:** Google Drive → **Download**  
- **File ID:** `={{ $json.id }}` (from `Pick Latest`)  
- **Binary Property:** `data` (default)  
- **Rename:** `Drive – Download CSV`

> ✅ After this, you have the **latest CSV** as **binary** on the `data` property.

---

## 3) Analyze — Parse CSV → compute KPIs → (LLM) executive summary
**Reminder (master rule):** LLM choice is **your preference** (OpenAI, Claude, Gemini, etc.).  
We’ll parse the CSV, compute KPIs in code, then **prompt** the LLM to write a summary.

### 3.1 Parse CSV to JSON
- **Node:** *Spreadsheet File* → **Read from file**  
- **Binary Property:** `data`  
- **Header row:** *Yes*  
- **Output:** *JSON*  
- **Rename:** `CSV → JSON`

### 3.2 Compute KPIs (Function)
- **Node:** *Function* → rename `Compute KPIs`
- **Code (paste):**
```js
// Input: array of rows from "CSV → JSON"
// Expected columns: date, order_id, product, category, units, revenue_inr, customer_id
const rows = items.map(i => i.json);

// Helpers
const num = v => Number(v ?? 0);

// Time window (this week) — purely informational here
const now = new Date();
const end = new Date(now.getFullYear(), now.getMonth(), now.getDate()); // today 00:00
const start = new Date(end); start.setDate(end.getDate() - 7);

let totalRevenue = 0;
let totalOrders = 0;
const byProduct = new Map();
const customers = new Set();

for (const r of rows) {
  const revenue = num(r.revenue_inr);
  const units = num(r.units);
  totalRevenue += revenue;
  totalOrders += 1;
  customers.add(String(r.customer_id || ''));

  const key = String(r.product || 'Unknown');
  const curr = byProduct.get(key) || { product: key, revenue: 0, units: 0, orders: 0 };
  curr.revenue += revenue;
  curr.units += units;
  curr.orders += 1;
  byProduct.set(key, curr);
}

const aov = totalOrders ? Math.round((totalRevenue / totalOrders) * 100) / 100 : 0;
const topProducts = [...byProduct.values()]
  .sort((a,b) => b.revenue - a.revenue)
  .slice(0, 5);

return [{
  json: {
    window_ist: {
      start: start.toISOString().slice(0,10),
      end: end.toISOString().slice(0,10)
    },
    kpis: {
      total_revenue_inr: totalRevenue,
      total_orders: totalOrders,
      unique_customers: customers.size,
      average_order_value_inr: aov,
      top_products: topProducts
    },
    raw_sample: rows.slice(0, 3) // keep a tiny sample for reference
  }
}];
