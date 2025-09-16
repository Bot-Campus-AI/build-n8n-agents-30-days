# Intelligent Email Assistant (n8n)

This package contains an import-ready workflow and screenshots.

## Import
- In n8n: **Workflows → Import from file** → choose `workflow/Intelligent Email Assistant.json`.
- After import, set credentials for:
  - **Gmail OAuth2** (Trigger + Reply)
  - **Google Sheets OAuth2**
  - **Gemini** (or swap to your model; Agent prompt is provider-agnostic)

## Flow
Gmail Trigger → Normalize Email (Set) → Sheet Id (Set) → Append row in sheet (Google Sheets) → AI Agent (LLM) → Code in JavaScript → Reply to a message (Gmail)

Keep the node names exactly as above (expressions reference labels).
