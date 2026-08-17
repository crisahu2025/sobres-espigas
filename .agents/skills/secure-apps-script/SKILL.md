---
name: secure-apps-script
description: >-
  Best practices and architectural patterns for securing Google Apps Script backends
  with static web frontends (GitHub Pages / SPA). Covers zero-client secrets,
  DevTools/Inspector leak prevention, server-side authentication, CORS, and robust token validation.
---

# Google Apps Script & Static Web Apps Security Standard

This skill establishes the security architecture and best practices when connecting static client-side web applications (such as GitHub Pages, Netlify, Vercel, or local HTML) to Google Apps Script backends.

---

## 🔒 1. The Zero-Client-Secret Principle

> **Core Rule:** NEVER store secrets, master keys, api tokens, or authentication maps inside frontend JavaScript (`.html`, `.js`, or client assets).

### Why?
Any user can open Google Chrome DevTools (F12 / Inspect / Network / Source) and immediately view all variables, constants, objects, and strings loaded in the browser.

### Proper Pattern:
1. **Frontend (`index.html`)**:
   - Contains **zero credentials**.
   - Accepts user input via `<input type="password">`.
   - Sends the user-supplied string directly to the backend over encrypted HTTPS (`POST` body or secure query parameter).
   - Only processes the boolean `success` / data returned by the backend.

2. **Backend (`Google Apps Script`)**:
   - Executes entirely on Google's private cloud infrastructure.
   - Holds the authority map of ministry/role passwords or hashes.
   - Evaluates authorization server-side:
     ```javascript
     function checkAuthorization(ministry, token) {
       if (!token) return false;
       const t = token.toString().trim();
       if (t === MASTER_SECURITY_TOKEN) return true;
       return MINISTRY_PASSWORDS[ministry] === t;
     }
     ```

---

## 🔑 2. Password Generation Standard for Ministry / Role Vaults

Avoid predictable incremental passwords (e.g., `MINISTERIO_2026`). Use non-deterministic entropy chunks:

```javascript
// High-Entropy Ministry Password Pattern:
// [PREFIX]-[RANDOM_ALPHANUMERIC_4]-[RANDOM_ALPHANUMERIC_4]
const MINISTRY_PASSWORDS = {
  "MINIST JOVENES": "J0V-7x9K-84m2",
  "MINIST DE ADOLESCENTES": "AD0-3p8W-51c7",
  "MINIST DE MUJERES": "MUJ-9v4R-62k3",
  "MINIST ESPECIALES": "ESP-8t2N-47x9",
  "MINIST DE EDUCACION CRISTIANA": "EDU-5m7Q-93j8",
  "MINIST DE PROTOCOLO": "PR0-6k3H-81w4"
};
```

---

## 🛡️ 3. Safe Endpoint Segregation in Apps Script

### Public vs Protected Operations:
- **Public Read (`action === 'getData'`)**: Returns non-sensitive schema data like ministry list and leader names for dropdown selectors.
- **Protected Read (`action === 'getHistorial'`)**: Requires `checkAuthorization(ministry, token)` before reading any spreadsheet rows.
- **Intake Writes (`doPost(e)`)**: Validates input structure, creates Google Drive Blobs, and appends rows to target sheet safely.

### Response Hardening:
- Always return standard JSON:
  ```javascript
  ContentService.createTextOutput(JSON.stringify(responseObject))
    .setMimeType(ContentService.MimeType.JSON);
  ```
- Reject unauthorized requests with explicit JSON errors rather than plain text or 200 OK leaks:
  ```javascript
  if (!isAuthorized) {
    return ContentService.createTextOutput(JSON.stringify({ 
      success: false, 
      error: "Clave de acceso incorrecta para el ministerio seleccionado." 
    })).setMimeType(ContentService.MimeType.JSON);
  }
  ```

---

## 🔄 4. Deployment Lifecycle in Google Apps Script

When updating security logic or password tables:
1. Save changes in the Apps Script Editor (`Ctrl + S`).
2. Navigate to **Deploy > Manage Deployments**.
3. Edit the existing Web App deployment (✏️ icon).
4. Under **Version**, choose **New Version**.
5. Click **Deploy**.
6. If the deployment URL updates, update `SCRIPT_URL` in the frontend.
