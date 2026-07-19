# SUMAT Eastvicians — Full-Stack Child Protection & Student Welfare Reporting System

A real client/server application: a Student Portal for submitting confidential reports and an
Admin Dashboard for 5 role-based accounts, sharing one PostgreSQL database through an Express API.

## Roles (v2 — child protection / student welfare model)

| Role | Type | Access |
|---|---|---|
| Principal | SUPER_ADMIN | Everything: all reports, all analytics, manage admin accounts, view audit logs, approve identity access requests |
| Child Protection Officer | CASE_MANAGER | All reports; investigates, updates status, assigns cases to the Counselor, may request identity access on Protected Identity reports |
| SSLG President | SYSTEM_MONITOR | Non-sensitive summaries and platform statistics only — never a reporter's identity, description, or notes |
| School Counselor | STUDENT_SUPPORT | Only cases explicitly assigned to them by the Principal or CPO; adds counseling notes, intervention status, follow-up dates |
| System Operator (Kian Jude) | Technical | Technical/platform administration only — no case-content access at all |

Access rules live in `backend/utils/permissions.js` and are enforced on every API call — a role
that shouldn't see a report can't get it even by calling the API directly, not just by a hidden UI button.

## Reporter privacy model

- **Confidential Report** — identity visible only to authorized officials (Principal, CPO); never
  shown to the accused or other students.
- **Protected Identity Report** — identity hidden behind a reporter alias (e.g. `Reporter #SUMAT-014`)
  until the Child Protection Officer requests access and the **Principal approves it**. Every
  request and decision is written to `identity_access_requests` and the audit log.

## Case workflow

```
Submitted → Under Review → Investigation Ongoing → Counseling/Intervention (if needed) → Action Taken → Closed
```

A case only enters "Counseling/Intervention" when the Principal or CPO explicitly assigns it to a
Counselor via `PATCH /api/reports/:id/assign-counselor`. The Counselor then updates their own
`counselor_status`/`counselor_notes`/`follow_up_date` fields through `PATCH /api/reports/:id/counselor`
— they cannot change the official investigation status.

## Why there's no student login

The original brief asked for both a student login system and a per-report verification form; this
build keeps the verification form (full name, grade, section, LRN captured per report) since that's
what's actually built into the HTML, rather than adding an unused second auth system. If you want
real student accounts (registration, "my reports" dashboard, cross-device tracking without typing a
Report ID each time), that's a separate, meaningful addition — let me know and I'll scope it.

## Local setup

### 1. Database
Create a PostgreSQL database (or use a free one from [neon.tech](https://neon.tech)). The schema
applies automatically the first time you run the seed script below.

### 2. Backend
```bash
cd backend
cp .env.example .env
# edit .env: set DATABASE_URL and a long random JWT_SECRET
npm install
npm run seed:admins   # applies the schema and creates the 5 accounts below
npm start              # API on http://localhost:5050
```

### 3. Frontend
Edit `frontend/config.js` so `API_BASE` points at your backend, then serve `frontend/` with any
static file server (or deploy it to Netlify — see below).

## Default administrator accounts

Created by `npm run seed:admins`. Change these before real-world use.

| Username | Password | Role |
|---|---|---|
| EVRSHS.Principal2026 | EVR@Admin26# | Principal |
| EVRSHS.CPO2026 | CPO_Safe26! | Child Protection Officer |
| SSLG.President26 | SSLG_Report26! | SSLG President |
| EVRSHS.Counselor2026 | Counselor.Safe26! | School Counselor |
| KianJude1 | kianjuden14344 | System Operator / Platform Administrator |

## Deployment

- **Backend**: needs a Node host — Render, Railway, Fly.io. Set `DATABASE_URL`, `JWT_SECRET`,
  `FRONTEND_ORIGIN`, `UPLOAD_DIR` as environment variables. A pure static host (Netlify alone)
  **cannot** run this — see the note below.
- **Frontend**: any static host — Netlify, Vercel, GitHub Pages. Update `frontend/config.js` with
  the deployed backend URL first.

### A note on "static-only" deployment
An earlier version of this brief asked for a fully static, backend-free system deployable straight
to Netlify. That's not compatible with the security requirements also in the brief (hashed
passwords, protected identities, tamper-resistant audit logs, cross-device dashboards) — a static
site's JavaScript is downloaded to every visitor's browser, so nothing in it can actually stay
confidential. This build keeps the real backend so those features are genuine, not cosmetic.

## Security notes

- Passwords are bcrypt-hashed (cost factor 12); never stored or transmitted in plain text after seeding.
- Admin sessions are JWTs (8-hour expiry) kept in memory in the browser tab, not `localStorage`.
- Evidence files live outside the public web root and are only downloadable through an
  authenticated, permission-checked endpoint.
- Login and report submission are rate-limited against brute-force and spam.
- Every report content query is filtered server-side by role and, for Protected Identity reports,
  by an approved identity-access request — enforced in the API, not just hidden in the UI.
- Audit log viewing is restricted to the Principal, per the spec.

## Known limitations / next steps

- Administrator account creation/editing happens via the seed script, not a UI, to avoid building a
  full user-management flow in this pass.
- Evidence files are stored on local disk in development; swap `backend/middleware/upload.js` for a
  cloud storage SDK before a real production deployment, especially on hosts with ephemeral disks.
- The multi-step reporting form (Step 1–5 wizard), the 4 separate dashboard UIs, and audit-log/chart
  visualizations described in the latest brief are not yet built into `frontend/` — the backend
  fully supports them (all the endpoints above exist and are tested), but the current HTML still
  uses the single-page form and one shared admin dashboard layout. Let me know if you want that
  frontend work done next.
