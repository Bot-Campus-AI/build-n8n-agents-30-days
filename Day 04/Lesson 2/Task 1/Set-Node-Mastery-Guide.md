#  n8n — Set Node Mastery: Transform, clean, and enhance


> **Auth note:** Set nodes need no auth. If you later add Google Sheets/Gmail, set up Google OAuth2 in n8n (Sheets/Gmail scope).

---

##  Goal
Take messy contact-form data (name, email, phone, message), **normalize & validate it** with a Set node, then route only **valid** items forward.

**Flow:** `Manual Trigger ➝ Set (Sample Input) ➝ Set (Clean & Enhance) ➝ IF (Valid?)`

---

##  Step-by-step

### 1) Manual Trigger — run anytime
- Drag **Manual Trigger** onto the canvas. No settings needed.

### 2) Set — Sample Input (simulate a form submit)
- Drag **Set** and rename to **Sample Input**.
- Toggle **Keep Only Set** = **On**.
- Add fields:
  - `fullName` = `  jAshwanth   boddupally  `
  - `email` = `  USER@Example.COM  `
  - `phone` = ` +91-93980 40588 `
  - `message` = ` Need a quote for website + chatbot `
  - `source` = `Website`

Run **Execute node** to emit this test item.

### 3) Set — Clean & Enhance (normalize, validate, enrich)
- Drag **Set** and rename to **Clean & Enhance**.
- Connect **Sample Input ➝ Clean & Enhance**.
- Toggle **Keep Only Set** = **On**.
- Add fields as **Expressions** (gear → *Add Expression*).

**A. Name (title-case)**
```js
={{ ($json.fullName || '')
  .trim()
  .toLowerCase()
  .split(/\s+/)
  .map(p => p.charAt(0).toUpperCase() + p.slice(1))
  .join(' ') }}
```

**B. Email (normalized)**
```js
={{ ($json.email || '').trim().toLowerCase() }}
```

**C. Email domain**
```js
={{ ($json.email || '').trim().toLowerCase().split('@')[1] || '' }}
```

**D. Phone (digits only)**
```js
={{ ($json.phone || '').replace(/\D/g, '') }}
```

**E. Phone (E.164, simple India logic)**
```js
={{ (()=>{ 
  const d = ($json.phone || '').replace(/\D/g,'');
  if (!d) return '';
  if (d.startsWith('91')) return '+' + d;
  if (d.length === 10) return '+91' + d;
  return '+' + d;
})() }}
```

**F. Message (trim)**
```js
={{ ($json.message || '').trim() }}
```

**G. Message length (number)**
```js
={{ (($json.message || '').trim()).length }}
```

**H. Valid email (boolean)**
```js
={{ /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(($json.email || '').trim()) }}
```

**I. Valid phone (boolean)**
```js
={{ (($json.phone || '').replace(/\D/g,'').length) >= 10 }}
```

**J. Masked email (safe preview)**
```js
={{ (()=>{ 
  const e = ($json.email || '').trim().toLowerCase();
  const [u,d] = e.split('@'); 
  if(!u || !d) return '';
  return u[0] + '***@' + d;
})() }}
```

**K. Tags (array)**
```js
={{ [
  ($json.source || 'unknown').toLowerCase(),
  ($json.email ? 'hasEmail' : 'noEmail'),
  ($json.phone ? 'hasPhone' : 'noPhone'),
  (($json.message || '').trim().length > 40 ? 'longMsg' : 'shortMsg')
] }}
```

**L. Timestamp (ISO string)**
```js
={{ $now.toISO() }}
```

### 4) IF — Valid?
- Drag **IF** and rename to **Valid?**.
- Connect **Clean & Enhance ➝ Valid?**.
- **Condition (Expression mode):**
```js
={{ $json.isValidEmail && $json.isValidPhone }}
```
- **True** output → proceed to databases/Sheets/CRM/email.  
- **False** output → send to a dead-letter or alert flow.

---

## Quick test
1. Execute the workflow.  
2. Inspect the **Clean & Enhance** output: normalized name/email/phone, flags, tags.  
3. Flip the input values (bad email/short phone) and confirm **IF** routing changes.

---

##  Expression quick-reference
- Trim & lower: `={{ ($json.email || '').trim().toLowerCase() }}`  
- Title-case: `={{ $json.fullName.toLowerCase().split(/\s+/).map(s=>s[0].toUpperCase()+s.slice(1)).join(' ') }}`  
- Digits only: `={{ ($json.phone || '').replace(/\D/g,'') }}`  
- Email regex: `={{ /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(($json.email||'').trim()) }}`  
- Tags array: `={{ [ $json.source, (($json.message||'').trim().length>40?'long':'short') ] }}`  
- Timestamp: `={{ $now.toISO() }}`

---

## Next steps (optional)
- **Google Sheets (Append):** map the clean fields to columns.  
- **Gmail/Slack:** notify sales with the enriched record.  
- **Dedup:** Add a query step (by email) before saving.
