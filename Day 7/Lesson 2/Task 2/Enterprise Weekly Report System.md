# 🏗️ Enterprise Weekly Report System — n8n Workflow

This workflow automatically collects data from multiple sources, processes it, generates executive insights, formats a beautiful HTML report, and distributes it to stakeholders **every Friday 4 PM IST**.

---

## ⏰ Step 1 — Trigger (Cron)
➝ **Drag & Drop ➝ Cron Node**

- **Mode:** `Custom`
- **Expression:** `0 16 * * 5`  
  (runs every Friday at 4 PM IST)
- **Rename:** `Trigger — Weekly Friday 4PM`

---

## 📥 Step 2 — Gather (Multi-source Data)

We connect **Google Sheets** as our foundation.

### 2.1 Create Google Sheets (manual prep)
Create these sheets in your Google Drive:

1. **Weekly Sales Data**  
   - Columns: `Date, Product, Revenue, Units, Salesperson`  
   - Add 4 weeks of sample rows.

2. **Marketing Metrics**  
   - Columns: `Date, Campaign, Impressions, Clicks, Conversions, Cost`  
   - Add 4 weeks of marketing campaign data.

3. **Project Tasks**  
   - Columns: `Week, Project, Tasks Completed, Hours Spent, Status`  
   - Add 4 weeks of project tracking.

4. **Financial Summary**  
   - Columns: `Week, Revenue, Expenses, Profit, Budget Variance`

💡 *Pro Tip*: Use fake but realistic numbers so testing feels real.

---

### 2.2 Collect Data (Google Sheets nodes)

➝ **Drag & Drop ➝ Google Sheets Node** (repeat for each sheet)

- **Operation:** `Read`
- **Range:** Entire sheet
- **Output:** JSON
- **Rename each:**
  - `Sales Data`
  - `Marketing Data`
  - `Project Tasks`
  - `Financial Summary`

---

## ⚙️ Step 3 — Process (Analytics Engine)

Now transform raw data into insights.

### 3.1 Compute KPIs
➝ **Drag & Drop ➝ Function Node** ➝ Rename: `Compute KPIs`

```js
// Combine data from multiple sheets
const sales = $items("Sales Data").map(i => i.json);
const marketing = $items("Marketing Data").map(i => i.json);
const projects = $items("Project Tasks").map(i => i.json);

// Sales KPIs
let totalRevenue = 0;
let totalOrders = 0;
let topPerformer = "";
let maxRevenue = 0;

for (const row of sales) {
  const revenue = Number(row.Revenue || 0);
  totalRevenue += revenue;
  totalOrders++;
  if (revenue > maxRevenue) {
    maxRevenue = revenue;
    topPerformer = row.Salesperson;
  }
}
const avgOrderValue = totalRevenue / totalOrders;

// Marketing KPIs
let totalSpend = 0, conversions = 0;
for (const m of marketing) {
  totalSpend += Number(m.Cost || 0);
  conversions += Number(m.Conversions || 0);
}
const costPerConversion = (totalSpend / conversions).toFixed(2);

// Project KPIs
const completed = projects.filter(p => p.Status === "Complete").length;
const totalTasks = projects.length;
const completionRate = ((completed / totalTasks) * 100).toFixed(1);

return [{
  json: {
    totalRevenue,
    avgOrderValue,
    topPerformer,
    totalSpend,
    conversions,
    costPerConversion,
    completed,
    totalTasks,
    completionRate
  }
}];
