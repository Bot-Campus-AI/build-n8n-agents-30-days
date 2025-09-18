# Guide.md — Bulletproof Error Handling with Google Sheets in n8n

⚠️ **Authentication Required**
- Google Sheets nodes require an OAuth2 credential in n8n. After importing the JSONs, open each Sheets node and select your credential.

---

## Power Pattern
**Manual Trigger ➝ Set Data ➝ Google Sheets (Try) ➝ IF Error? ➝ (Recover path) Create Spreadsheet ➝ Append (Recover)**

---

## Deliverables
-  — intentionally broken Google Sheets append (to demonstrate failure).
-  — captures the error, creates a new Sheet, then appends successfully.

---


## Walkthrough

### A) ❌ Error workflow — *GS Error Demo - Append Fails*
**Flow ➝** Manual Trigger ➝ Set Data ➝ Set Bad Config ➝ Google Sheets: Append (fails)

- **Set Data**: prepares sample fields `Name`, `Email`.
- **Set Bad Config**: injects a fake Spreadsheet ID (`1INVALID_ID_DEMO_NOT_FOUND`) and `range = Sheet1!A:B`.
- **Google Sheets - Append (Will Error)**:
  - Operation = *Append*
  - documentId = `={{$json.documentId}}`
  - range = `={{$json.range}}`
  - valueInputMode = RAW
  - On Execute: Google rejects with *“Requested entity was not found”*.

### B) ✅ Error-handled workflow — *GS Error Handling Demo - Recover by Creating Sheet*
**Flow ➝** Manual Trigger ➝ Set Data ➝ Set Bad Config ➝ **Google Sheets - Append (Try)** *(Continue On Fail = true)* ➝ **IF Error?** ➝
- **True path** (no error): **Set OK**.
- **False path** (error present): **Create Spreadsheet** ➝ **Set Data (Retry Payload)** ➝ **Append (Recover)**.

**Key settings**
- **Append (Try)**: `continueOnFail: true` — doesn’t stop execution; emits an `error` property if Google rejects.
- **IF Error?**: checks `={{$json.error}}` *isEmpty*:
  - If empty ➝ ok path.
  - If not empty ➝ recovery path.
- **Append (Recover)**: uses the newly created ID  
  `={{$node["Google Sheets - Create Spreadsheet"].json["spreadsheetId"]}}`

---

## Verify
- Run the **Error** workflow — you should see the Sheets node stop with an error.
- Run the **Error-handled** workflow — it will branch to recovery and append to a brand-new Sheet.

---

## Real-World Tips
- For not-found IDs, recover by **creating** the resource or **selecting** a fallback.
- For rate limits (429), add a **Wait** + **Retry** loop.
- Always log `error.message` to a Slack/Email node for visibility in production.

**End of Guide**
