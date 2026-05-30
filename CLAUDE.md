# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **Sala-Repair** (ระบบแจ้งซ่อมบำรุง) — a repair/maintenance ticketing system for บริษัท ศาลาโอสถรีเทล จำกัด. It is a two-file project with no build system:

- `index.html` — the entire frontend SPA (HTML + vanilla JS + Tailwind CDN + Lucide icons)
- `apps-script.gs` — Google Apps Script backend deployed as a Web App (not run locally)

The app is deployed as a static site at `https://goodyearzph.github.io/salaosot-repair-system/`.

## Local Development

There is no package manager or build step. Serve the file directly:

```bash
python3 -m http.server 3000
# or
npx serve -p 3000
```

Then open `http://localhost:3000`. The Google OAuth client is configured to allow `http://localhost:3000` as a JavaScript origin.

If the Google OAuth client ID is not working, use the **dev bypass button** that appears in the login screen setup notice — it calls `devBypass()` which logs in as a test user without Google auth.

## Architecture

### Frontend (`index.html`)

**Configuration block** (lines ~52–77): Four constants must be set at the top of the file before deploying:
- `GOOGLE_CLIENT_ID` — OAuth 2.0 Web App client ID from Google Cloud Console
- `ADMIN_EMAILS` — hardcoded array of super-admin emails (cannot be removed from the app's Settings tab)
- `APPS_SCRIPT_URL` — deployed Apps Script Web App URL
- `BRANCH_LIST` — array of branch name strings

**Two storage modes**: The boolean `useSheets` is set in `enterApp()` based on whether `APPS_SCRIPT_URL` is configured. When `true`, all reads/writes go to the Apps Script API. When `false`, data falls back to `localStorage`.

**Authentication flow**: Google Identity Services (GSI) renders a sign-in button. After the credential JWT is decoded in `handleCredentialResponse()`, `enterApp()` runs: it sets `useSheets`, loads dynamic admins and tech emails from Sheets, then determines `isAdmin` (super admin OR in dynamic list).

**Three roles**:
1. Regular user — sees "แจ้งซ่อม" tab only; can only see their own tickets (filtered by email)
2. Admin — sees "แดชบอร์ด" tab; can update status, add notes, delete tickets, generate PDF reports
3. Super Admin — also sees "จัดการแอดมิน" tab; can add/remove dynamic admins and tech emails

**Ticket data shape**:
```js
{ id, name, email, branch, phone, equip, deviceId, category, desc, urgency,
  images, status, note, createdAt, updatedAt }
```
- `id` is `Date.now()` (13-digit timestamp)
- `status`: `'waiting'` | `'progress'` | `'done'`
- `urgency`: `'low'` | `'medium'` | `'high'`
- `images`: array of Google Drive thumbnail URLs (or base64 data URLs in localStorage mode)

**Overdue logic** (`isOverdue`): tickets not yet `'done'` that exceeded `{ high: 2, medium: 7, low: 30 }` days since `createdAt`.

### Backend (`apps-script.gs`)

Deployed as a Google Apps Script Web App ("Execute as: Me, Anyone can access"). The script operates on the active Google Spreadsheet which must have three sheets: `Tickets`, `AdminEmails`, `TechEmails`. These are auto-created on first use.

**`doGet(e)`** — reads data:
- Default (no `action`): returns all rows from `Tickets` sheet as JSON
- `?action=getAdmins` — returns emails from `AdminEmails` sheet
- `?action=getTechs` — returns emails from `TechEmails` sheet

**`doPost(e)`** — writes data (body is JSON in `e.postData.contents`):
- `add` — appends ticket row, then calls `sendNewTicketNotification()`
- `uploadImage` — decodes base64 image, saves to Google Drive folder `RepairTicketImages`, returns public thumbnail URL
- `update` — updates a single field in the matching ticket row by `id`; triggers `sendCompletionNotification()` when `field=status, value=done`
- `delete` — deletes the matching row
- `clear` — deletes all data rows
- `addAdmin` / `removeAdmin` — manages `AdminEmails` sheet
- `addTech` / `removeTech` — manages `TechEmails` sheet

**Email notifications** use `MailApp.sendEmail()`:
- On new ticket: tech emails get details; requester gets a confirmation
- On status → `done`: requester gets completion notice; tech emails also notified

## Deploying Apps Script Changes

`apps-script.gs` is not run by any local toolchain. To deploy changes:
1. Open the linked Google Apps Script project
2. Paste the updated code
3. Deploy > Manage Deployments > create a **New Deployment** (do not edit existing — old URL breaks)
4. Update `APPS_SCRIPT_URL` in `index.html` with the new deployment URL

## Key Conventions

- All user-facing text is in Thai. Keep it that way.
- `escHtml()` must wrap any user-supplied string before inserting it into `innerHTML`.
- After any DOM manipulation that inserts Lucide icon elements, call `lucide.createIcons()`.
- The `useSheets` flag must be set before `loadDynamicAdmins()` and `loadTechEmails()` are called — the order in `enterApp()` is intentional (fixed in commit `915a27b`).
- Ticket IDs use the last 5 digits for display (`String(t.id).slice(-5)`) and last 8 digits in documents/emails.
