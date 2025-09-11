# 📝 n8n Form + Google Sheets — Daily Activities Logger

This workflow demonstrates how to collect daily activities using an **n8n Form**, process them, and append results to **Google Sheets** for tracking and analysis.

---

## 🔑 Authentication Note
- **Google Sheets node** requires OAuth2 authentication with Google.  
- Make sure you have a Google Sheet ready with these columns:  
  **Date | Activity | Hours Spent | Fun Level (1-10)**  

---

## ⚙️ Workflow Steps

### 1 ➝ Form Trigger (Collect Activity Input)
- Drag & drop **Form Trigger** node → rename to **Daily Activity Form**.  
- Configure form fields:
  - `date` (Date input)  
  - `activity` (Text input)  
  - `hours_spent` (Number input)  
  - `fun_level` (Dropdown: 1–10)  

👉 This will generate a form URL you can share to submit data.

---

### 2 ➝ Extract Details
- Add **Set** node → rename to **Extract Details**.  
- Connect ➝ `Daily Activity Form ➝ Extract Details`.  
- **Keep Only Set:** ON.  
- Fields:
  - `date` = `={{ $json.date }}`  
  - `activity` = `={{ $json.activity }}`  
  - `hours_spent` = `={{ $json.hours_spent }}`  
  - `fun_level` = `={{ $json.fun_level }}`  

---

### 3 ➝ Standardize Data (Set Node)
- Add another **Set** node → rename to **Standardize Data**.  
- Connect ➝ `Extract Details ➝ Standardize Data`.  
- Purpose: ensure consistent formatting before saving.  
- Fields:
  - `date` = `={{ new Date($json.date).toLocaleDateString("en-GB") }}`  
  - `activity` = `={{ $json.activity.trim() }}`  
  - `hours_spent` = `={{ Number($json.hours_spent) }}`  
  - `fun_level` = `={{ Number($json.fun_level) }}`  

---

### 4 ➝ Google Sheets: Append Data
- Add **Google Sheets** node → rename to **Append Activity**.  
- Connect ➝ `Standardize Data ➝ Append Activity`.  
- Configure:
  - **Operation:** `Append`  
  - **Authentication:** Google OAuth2  
  - **Spreadsheet URL:** link to your Google Sheet  
  - **Range:** `Sheet1!A:D`  
- Map columns:
  - A → Date  
  - B → Activity  
  - C → Hours Spent  
  - D → Fun Level  

---

## 📄 Sample Daily Activities Sheet

| Date     | Activity      | Hours Spent | Fun Level |
|----------|---------------|-------------|-----------|
| Jan 15   | Reading Book  | 2           | 8         |
| Jan 15   | Gym Workout   | 1.5         | 9         |
| Jan 15   | Cooking       | 1           | 7         |
| Jan 16   | Office Work   | 8           | 6         |
| Jan 16   | Movie Night   | 3           | 10        |
| Jan 16   | Dinner Party  | 2.5         | 9         |
| Jan 17   | Cycling       | 1.5         | 8         |
| Jan 17   | Study Session | 4           | 7         |
| Jan 17   | Gaming        | 2           | 9         |
| Jan 18   | Morning Walk  | 1           | 8         |
| Jan 18   | Coding Project| 6           | 9         |

---

## 🎯 What You Can Do Next
- Calculate **total hours** across all rows.  
- Calculate **average fun level**.  
- Build analytics dashboards with Sheets or connect to another tool (Looker Studio, Slack, etc.).

💡 Easy Start: You can copy/paste this table into a new Google Sheet and then use the n8n workflow to append more rows dynamically.

---
