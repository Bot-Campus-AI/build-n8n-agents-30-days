# 📊 n8n Google Sheets Challenge — Track & Analyze Your Data

This project will help you practice **Google Sheets integration in n8n**.  
We’ll fetch data from a Google Sheet (daily activities/expenses), clean it, and calculate simple analytics.

---

## 🔑 Authentication Note
Before starting, you must:
- Enable **Google Sheets API** in your Google Cloud project.
- Connect n8n with **Google OAuth2 credentials** (Sheets scope).

---

## ⚙️ Workflow Steps

### 1 ➝ Manual Trigger
- Drag & drop **Manual Trigger** node.  
- Purpose: start the workflow on demand.

---

### 2 ➝ Google Sheets Node: Read Data
- Drag & drop **Google Sheets** node → rename to **Read Sheet**.  
- Connect ➝ `Manual Trigger ➝ Read Sheet`.  
- Configure:
  - **Operation:** `Read`  
  - **Authentication:** Google OAuth2 (your connected account)  
  - **Spreadsheet URL:** paste the Google Sheet link (your sheet with columns `Date | Item | Details | Notes | Amount | Category`)  
  - **Range:** select the full table (e.g., `Sheet1!A:F`)  

📄 **Sample Sheet Data**

| Date    | Item            | Details                 | Extra Notes          | Amount | Category       |
|---------|-----------------|-------------------------|----------------------|--------|----------------|
| Jan 15  | Lunch           | Veg Thali               | With colleagues      | 12.50  | Food           |
| Jan 15  | Dinner          | Biryani                 | Takeaway             | 18.00  | Food           |
| Jan 15  | Tiffin          | Masala Dosa             | Evening snack        | 6.50   | Food           |
| Jan 16  | Movie Ticket    | Cinema Hall             | Popcorn + Drinks     | 15.00  | Entertainment  |
| Jan 16  | Petrol          | Car Refuel              | 5 Liters             | 40.00  | Transport      |
| Jan 17  | Book            | Python Programming      | Technical Reference  | 8.99   | Education      |
| Jan 17  | Files & Stationery | Office Supplies     | Documents + Folders  | 12.75  | Education      |
| Jan 18  | Gym Membership  | Monthly Subscription    | Fitness Center       | 25.00  | Health         |
| Jan 18  | Coffee          | Starbucks Cappuccino    | Morning routine      | 4.50   | Food           |
| Jan 18  | Uber Ride       | Office to Home          | Evening commute      | 10.00  | Transport      |

---

### 3 ➝ Set Node: Extract Fields
- Drag & drop **Set** node → rename to **Extract Fields**.  
- Connect ➝ `Read Sheet ➝ Extract Fields`.  
- **Keep Only Set:** ON.  
- Fields (map columns to clean JSON):
  - `date` → `={{ $json["Date"] }}`
  - `item` → `={{ $json["Item"] }}`
  - `details` → `={{ $json["Details"] }}`
  - `notes` → `={{ $json["Extra Notes"] }}`
  - `amount` → `={{ parseFloat($json["Amount"]) }}`
  - `category` → `={{ $json["Category"] }}`

---

### 4 ➝ Function Node: Calculate Totals
- Drag & drop **Function** node → rename to **Calculate Totals**.  
- Connect ➝ `Extract Fields ➝ Calculate Totals`.  
- Paste code:

```js
let totalSpent = 0;
let itemCount = 0;
let categoryTotals = {};

for (const row of items) {
  const amount = row.json.amount || 0;
  totalSpent += amount;
  itemCount++;
  const category = row.json.category || "Uncategorized";
  if (!categoryTotals[category]) categoryTotals[category] = 0;
  categoryTotals[category] += amount;
}

return [{
  json: {
    totalSpent,
    itemCount,
    avgPerItem: totalSpent / (itemCount || 1),
    categoryTotals
  }
}];
```

---

### 5 ➝ IF Node: Spending Check
- Drag & drop **IF** node → rename to **Check Overspend**.  
- Connect ➝ `Calculate Totals ➝ Check Overspend`.  
- Condition:
  - **Value 1:** `={{ $json.totalSpent }}`  
  - **Operation:** `larger`  
  - **Value 2:** `100`  

**True path** → overspending alert.  
**False path** → normal processing.

---

### 6 ➝ Gmail Node: Send Report
- Add **Gmail** node → rename to **Send Report**.  
- Connect ➝ `Check Overspend (true)`  
- Configure:
  - **To:** your email  
  - **Subject:** `Daily Expense Report`  
  - **Body:**  
    ```html
    <h3>📊 Daily Expense Summary</h3>
    <p><b>Total Spent:</b> ${{$json.totalSpent}}</p>
    <p><b>Items:</b> {{$json.itemCount}}</p>
    <p><b>Average per Item:</b> ${{$json.avgPerItem}}</p>
    <h4>Category Breakdown</h4>
    <pre>{{$json.categoryTotals}}</pre>
    ```

---

## ✅ Decision Logic
- **Total > $100** → send overspend alert via Gmail  
- **Total ≤ $100** → no action (normal case)  

---

## 🎯 Try It Yourself
1. Create a Google Sheet with **daily activity or expense rows**.  
2. Add columns: Date, Item, Details, Notes, Amount, Category.  
3. Insert 5–10 rows of sample data.  
4. Import this workflow JSON into n8n.  
5. Run it and check the output (inspect totals + Gmail email).  
