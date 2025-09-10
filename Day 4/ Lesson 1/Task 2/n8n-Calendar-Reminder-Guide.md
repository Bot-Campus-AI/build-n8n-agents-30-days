# 📅 n8n Calendar Connections — Daily Google Calendar → Gmail Reminder (IST)

> ⚠️ **Authentication first (required):**
> - **Google Calendar**: Create a **Google OAuth2** credential in n8n and authorize access with **Calendar** scope.
> - **Gmail**: Create a **Google OAuth2** credential in n8n and authorize access with **Gmail** scope.
> - If your Google Cloud app isn’t verified, add your account as a **Test User** on the OAuth consent screen to avoid “Access blocked” errors.

---

## 🎯 What this workflow does
Every **15 minutes (09:00–20:00 IST)**, it looks for events starting in the **next 15 minutes** on your **Primary Google Calendar**.  
For each upcoming event, it **sends a reminder email** via **Gmail** to the organizer (or a default address) with the event title, time, and Meet/Zoom link (if present).

**Flow:** `Cron (15m) ➝ Date & Time (window) ➝ Google Calendar (Get many) ➝ Split In Batches ➝ Gmail (send reminder)`

... (content trimmed for brevity in this code block) ...
