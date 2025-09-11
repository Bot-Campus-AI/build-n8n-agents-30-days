# 📝 n8n Form ➝ Google Sheets — Daily Expense Tracker

This workflow lets you **submit expenses via an n8n form** and automatically **append them to a Google Sheet**.  
You’ll track `Date`, `Item`, `Amount`, and `Category` — then later you can analyze totals and averages.

---

## 🔑 Authentication Note
- **Form Node:** No auth needed.  
- **Google Sheets Node:** Requires Google OAuth2 connection (enable Google Sheets API in Google Cloud).

---

## ⚙️ Workflow Steps

### 1 ➝ Form Node: Collect Expense Data
- Drag & drop **Form Trigger** node → rename to **Expense Form**.  
- Configure:
  - **Fields:**
    - `date` (Date Picker)  
    - `item` (Text Input)  
    - `amount` (Number Input)  
    - `category` (Dropdown: Food, Entertainment, Education, Transport, Health, Other)  

➡️ This creates a public form URL where you can enter daily expenses.

---

### 2 ➝ Set Node: Clean Data
- Drag & drop **Set** node → rename to **Clean Expense Data**.  
- Connect ➝ `Expense Form ➝ Clean Expense Data`.  
- **Keep Only Set** = ON.  
- Fields:
  - `date` = `={{ $json["date"] }}`
  - `item` = `={{ $json["item"] }}`
  - `amount` = `={{ parseFloat($json["amount"]) }}`
  - `category` = `={{ $json["category"] }}`

---

### 3 ➝ Google Sheets Node: Append Row
- Drag & drop **Google Sheets** node → rename to **Append to Sheet**.  
- Connect ➝ `Clean Expense Data ➝ Append to Sheet`.  
- Configure:
  - **Operation:** `Append`  
  - **Authentication:** Google OAuth2 (your account)  
  - **Spreadsheet URL:** link to your Google Sheet  
  - **Sheet Name:** e.g., `Expenses`  
  - **Columns Mapping:**
    - Date → `{{$json["date"]}}`
    - Item → `{{$json["item"]}}`
    - Amount → `{{$json["amount"]}}`
    - Category → `{{$json["category"]}}`

---

## 📄 Sample Google Sheet

| Date    | Item          | Amount | Category      |
|---------|---------------|--------|---------------|
| Jan 15  | Lunch         | 12.50  | Food          |
| Jan 15  | Dinner        | 18.00  | Food          |
| Jan 15  | Tiffin        | 6.50   | Food          |
| Jan 16  | Movie Ticket  | 15.00  | Entertainment |
| Jan 16  | Petrol        | 40.00  | Transport     |
| Jan 17  | Book          | 8.99   | Education     |
| Jan 17  | Files & Stationery | 12.75 | Education |
| Jan 18  | Gym Membership | 25.00 | Health        |
| Jan 18  | Coffee        | 4.50   | Food          |
| Jan 18  | Uber Ride     | 10.00  | Transport     |

---

## ✅ How It Works
- You open the n8n form ➝ fill in Date, Item, Amount, Category.  
- n8n **cleans the values** with the Set node.  
- n8n **appends the data** to your Google Sheet in the correct row.  
- You can later extend the workflow with:
  - **Function node** ➝ calculate totals, averages.  
  - **Gmail/Slack node** ➝ send daily/weekly reports.  

---

## 🎯 Try It Yourself
1. Create a Google Sheet named `Expenses` with columns `Date | Item | Amount | Category`.  
2. Import this workflow JSON into n8n.  
3. Fill the form a few times (Lunch, Movie, Book, Coffee).  
4. Check your sheet ➝ new rows appear automatically.  
