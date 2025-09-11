# 📊 n8n Google Sheets Form Workflow — Daily Activities Logger

This workflow demonstrates how to collect data via an **n8n Form**, extract details, and append them into a **Google Sheet** for analysis.

---

## 🔑 Authentication Note
- The **Google Sheets** node requires **Google OAuth2 credentials** in n8n.  
- Ensure your Google Sheet is created with columns:  
  - **Date | Activity | Hours Spent | Fun Level (1-10)**

---

## ⚙️ Workflow Steps

### 1 ➝ Form Node: Collect Daily Activity
- Drag & drop **Form** node → rename to **Daily Activity Form**.  
- Connect ➝ `Manual Trigger ➝ Daily Activity Form` (optional: skip trigger if form auto-generates link).  
- Configure fields:
  - `date` → Date Picker  
  - `activity` → Text Input  
  - `hours` → Number Input  
  - `fun_level` → Dropdown (1–10)  

📄 **Example Form Fields**  
- Date: `2025-09-11`  
- Activity: `Morning Jog`  
- Hours Spent: `1.5`  
- Fun Level: `8`

---

### 2 ➝ Extract Details
- Drag & drop **Set** node → rename to **Extract Details**.  
- Connect ➝ `Daily Activity Form ➝ Extract Details`.  
- **Keep Only Set** = ON.  
- Fields:
  - `date` = `={{ $json["date"] }}`  
  - `activity` = `={{ $json["activity"] }}`  
  - `hours` = `={{ $json["hours"] }}`  
  - `fun_level` = `={{ $json["fun_level"] }}`  

---

### 3 ➝ Set Node: Format Data
- Drag & drop **Set** node → rename to **Format Data**.  
- Connect ➝ `Extract Details ➝ Format Data`.  
- Purpose: ensure values are clean before appending.  
- Fields:
  - `Date` = `={{ $json["date"] }}`  
  - `Activity` = `={{ $json["activity"] }}`  
  - `Hours Spent` = `={{ parseFloat($json["hours"]) }}`  
  - `Fun Level` = `={{ parseInt($json["fun_level"]) }}`  

---

### 4 ➝ Google Sheets Node: Append to Sheet
- Drag & drop **Google Sheets** node → rename to **Append Activity**.  
- Connect ➝ `Format Data ➝ Append Activity`.  
- Configure:
  - **Operation:** `Append`  
  - **Authentication:** Google OAuth2 (your account)  
  - **Spreadsheet URL:** paste your Google Sheet link  
  - **Range:** select `Sheet1!A:D`  
  - **Values:** map fields from `Format Data` → (Date, Activity, Hours Spent, Fun Level)  

---

## 📄 Sample Google Sheet
After running the workflow, your Google Sheet might look like this:

| Date       | Activity       | Hours Spent | Fun Level |
|------------|----------------|-------------|-----------|
| 2025-09-11 | Morning Jog    | 1.5         | 8         |
| 2025-09-11 | Work Project   | 6           | 7         |
| 2025-09-11 | Dinner w/Family| 2           | 9         |
| 2025-09-12 | Reading Book   | 1           | 6         |
| 2025-09-12 | Movie Night    | 2.5         | 10        |

---

## 🧮 Next Step: Simple Analytics
You can later add a **Function node** to calculate:
- Total Hours in a Day → `sum(hours)`  
- Average Fun Level → `avg(fun_level)`  

Or use **Google Sheets formulas** to compute directly inside the sheet.

---

## ✅ Try It Yourself
1. Create a Google Sheet with columns: Date, Activity, Hours Spent, Fun Level.  
2. Import this workflow into n8n.  
3. Submit entries through the **n8n Form link**.  
4. Verify new rows are added to your sheet.  
5. Expand workflow with a Function/Gmail node to send yourself daily reports.

---
