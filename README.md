# IT Portal - Service Desk + IT Claims (prototype)

One portal for IT:
- **Service desk (ITSM)**: tickets, approval lines, auto-allocation to the best-fit engineer, SLA clock with pause, escalating SLA alerts, KPI dashboards.
- **IT claims**: upload IT invoices / receipts (hardware, software, services, telecom, cloud). Approval goes Manager → HOD → VP or C-suite → Finance → Paid. A claim can be linked to a ticket.

| Part | Tech | Port |
|---|---|---|
| `backend/` | Django 6.1 + Django REST Framework, SQLite | 8100 |
| `frontend/` | Next.js (App Router, strict TypeScript) + shadcn/ui on Tailwind v4, proxies `/api/*` to the backend | 3000 |

UI theme: dark by default with ocean-blue accents. Each user can pick Dark or Light under Settings (account menu); the choice is saved to their account and applied when they sign in. The theme tokens are in `frontend/app/globals.css`.

The full REST contract is in [API.md](API.md).

## Run (Windows)

```
# one-time
python -m venv .venv
.venv\Scripts\pip install -r backend\requirements.txt
.venv\Scripts\python backend\manage.py migrate
.venv\Scripts\python backend\manage.py seed_demo          # fictional demo data (--reset to start over)
# the demo has ~4 months of history: 450 closed tickets with timelines, ~150 claims, a month of sign-ins
# (--small gives the small set the tests use)
# every demo claim has a generated sample invoice / receipt PDF, watermarked SAMPLE - DEMO ONLY (fictional data)
# demo sign-in: user IDs E1001-E1026 (or their emails) with the shared demo password,
# DEFAULT_DEMO_PASSWORD in backend/core/management/commands/seed_demo.py (override with PORTAL_DEMO_PASSWORD)
cd frontend && npm install && cd ..

# each time (two terminals)
.venv\Scripts\python backend\manage.py runserver 127.0.0.1:8100
npm --prefix frontend run dev                              # open http://localhost:3000
```

Tests: `.venv\Scripts\python backend\manage.py test`

Frontend checks: `npm --prefix frontend run typecheck` and `npm --prefix frontend run lint`. API types live in `frontend/lib/types.ts` (keep in sync with `API.md`).

## Service desk flow

```
Employee raises ticket: Incident (something broken) or Request (something new)
  -> picks Level 1 -> Level 2 from the service catalog, e.g. Request > Software > Git
  -> approval line (per Level 2 item: LINE_MANAGER -> POOL_MANAGER -> HOD, any subset, or none)
  -> IT pool (ITHQ / ITAA / ITBA)  SLA clock starts
  -> auto-allocation to best-fit member (skill, load, past SLA performance, fairness; priority-adaptive)
  -> pick up (= response SLA) -> In progress <-> On hold / Pending user (clock paused)
  -> Resolved -> IT manager confirms SLA met / not met -> Closed (or reopen)
```
Incidents are numbered `INC000123`, requests `REQ000045`.

**Service catalog** (Service Desk > Service catalog; Head of IT and IT managers): incidents and requests each have their own Level 1 > Level 2 tree. A Level 2 item decides the routing: IT pool, skill used by the allocator, default priority and approval line. Items and groups are deactivated, never deleted, so old tickets keep their path.

The SLA monitor checks every 30 s. It alerts at 75 % / 90 % / breach and re-allocates tickets that nobody picked up in time. In development it runs inside `runserver`. In production run `manage.py sla_monitor` as its own process and set `SLA_MONITOR_IN_PROCESS=0`.

## IT claims flow

```
Submit (invoice or receipt + PDF/PNG/JPG, optional ticket ref)
  -> Manager -> HOD -> VP or C-suite -> Finance   each can Approve / Return for amendment / Reject
  -> Approved (awaiting payment) -> Finance records payment ref -> Paid
Returned -> requester edits, adds/removes files, resubmits -> chain restarts (new round)
```
- **Who approves**: Manager = the requester's manager. HOD = the requester's department HOD. Above the HOD: the department's VP, or its C-suite approver when the department has no VP (e.g. a department that reports straight to the CEO). Finance = any finance user.
- **Skips**: a step is skipped when it has no approver, the approver is the requester, or the same person approved an earlier step.
- **Chain start**: the chain starts where the requester sits. An executive's claim goes Manager → HOD → VP or C-suite, a manager's claim starts at the HOD, and an HOD's claim goes straight to the VP or C-suite (no one approves their own claim).
- **HOD limits** (Finance HOD only, on the claims Settings page): MYR claims at or below the limit are final at the HOD and skip the VP or C-suite step. Non-MYR claims always go to the VP or C-suite. Every change is logged.
  - One **general** limit (default MYR 5,000), plus optional **individual** limits for a particular HOD, or for a VP or C-suite approver (covering all their departments).
  - For a claim, the requester's HOD's individual limit is used first, then their VP's or C-suite's, then the general limit.
  - `-1` means no limit: every claim needs VP or C-suite approval.
  - HOD limits and the general limit apply at once. **VP and C-suite limits need CFO approval** (Limit approvals page): they apply only after the CFO approves. The CFO is the VP (or C-suite) of the Finance HOD's department; the CFO's own limit cannot be set individually, since they cannot approve it themselves.
- **Controls**:
  - No one approves or pays their own claim.
  - Same vendor + document no. on an open claim gets a duplicate warning (HTTP 409; the user can confirm and submit anyway).
  - Files are type-checked by their content, max 5 × 10 MB.
  - Files download only through a permission check.
  - Removed files are kept for audit.

## Sign in

- Users sign in with their **user ID** (e.g. `E1001`) or **email**, plus a password. Users without an email address sign in with their user ID.
- Passwords are stored only as Django password hashes. New passwords need at least 10 characters, must not be common or only numbers, and must not resemble the user's ID, name or email. Users change their password under **Settings**; doing so signs out their other devices.
- After 5 wrong passwords the account locks for 15 minutes. The error message is the same for unknown users, so it does not reveal which accounts exist.
- The session is an HttpOnly cookie (scripts cannot read it), ends after 8 h idle or 24 h, and is removed on **Sign out**. Changes must carry the `X-Requested-With` header the portal sends, which blocks cross-site request forgery.
- Sign-ins, failures, lockouts and password changes are in the audit log.

## User accounts (Administration > Users, system admin)

- Add and edit users: user ID, name, email (optional), role, department, manager and IT pool (IT roles).
- New users and password resets get a **temporary password**, shown to the admin once. The user must choose their own password at the next sign-in, and cannot do anything else until then. A reset also unlocks the account and signs it out everywhere.
- **Unlock** locked accounts; **deactivate / reactivate** users (deactivating signs them out everywhere; history is kept).
- Safety: unique user IDs and emails, no manager loops, admins cannot deactivate or demote themselves, and at least one active admin always remains.
- Department heads (HOD / VP / C-suite of each department) are not edited here yet.

## Roles

| Role | Service desk | IT claims |
|---|---|---|
| Employee | Raise & track tickets, approve direct reports' tickets | Submit & track claims; approve if someone's manager / HOD |
| IT member | Queue sorted by SLA, pick up / hold / resolve, self-pick from pool, team KPI | Submit claims (e.g. parts, licences) |
| IT manager | Pool dashboard, SLA confirmation, reassign, team skills/capacity/leave, pool KPIs | Approve team members' claims (Manager step) |
| Head of IT (hod) | All pools, IT-wide KPIs, SLA policy, allocation weights | HOD step for IT claims |
| VP | - | VP step for their departments |
| C-suite | - | Top step for departments without a VP |
| CFO (VP of Finance) | - | VP step for Finance; approves VP and C-suite limit changes |
| Finance | - | Finance step, payments, all claims & totals; Finance HOD sets the general and individual HOD limits |
| System admin | System settings (allocation weights, SLA alerts, monitor interval); audit log view and CSV export | Submit & track own claims. Cannot change the HOD limits (Finance settings) |

## Email notifications

- **SMTP** (Administration > System settings, admin only): server, port, encryption (STARTTLS / SSL / none), login, sender and the portal address used for links. Includes a test-email button and a delivery status (sent, queued, failed, last error).
  - The SMTP password is encrypted in the database and never sent back to the browser or written to the audit log. The key comes from `PORTAL_ENCRYPTION_KEY` (a Fernet key); in development a key is created in `backend/portal.fernet.key` (git-ignored). If the key changes, re-enter the password.
- **Settings** (account menu, every user): Appearance (Dark / Light, default Dark) and Notifications - for each notification type, choose In portal (the bell) and/or Email. Users only see the types that apply to their role.
- Emails are queued and sent by a background sender every 10 s (inside `runserver` in development, or `manage.py sla_monitor` in production), retried up to 5 times. A slow or broken mail server never slows down the portal.

## Lists: filters and pages

Every list (my tickets, pool queue, tickets decided by me, my claims, claim approvals, claims decided by me, payments, all claims, audit log) has the same filter bar: search, time range (all time, last 24 hours, 3 days, 7 days, 30 days, or custom dates) and, where it fits, status, priority or source. Click a column header to sort the whole list (click again to reverse); the arrow shows the order. Filters and sort are kept for the browser session and any change returns to page 1. Lists show 20 rows per page by default (10 to 100).

## Audit log

The system admin sees ticket, claim and system events in one list (Administration > Audit log), filtered by source, dates and text, and can export them to CSV. System events are setting, SLA and SMTP changes, test emails, HOD limit changes and requests (with the CFO's decision) and the audit exports themselves. CSV cells that start with `=`, `+`, `-` or `@` are prefixed with `'` so spreadsheets do not run them as formulas.

## Before production

- **Accounts**: the demo users share one demo password - reseed without demo data, or set real passwords, before any real use. Admins reset passwords and unlock accounts under Administration > Users. Consider company SSO (e.g. Microsoft Entra ID via OIDC) and loading the org chart (manager, department HOD / VP / C-suite, IT pools) from HR data.
- **HTTPS is required** in production so the `Secure` session cookie is sent (`DJANGO_DEBUG=0`).
- **Server settings**: set `DJANGO_SECRET_KEY`, `DJANGO_DEBUG=0` and `DJANGO_ALLOWED_HOSTS`. Run behind HTTPS with gunicorn/uvicorn, not `runserver`.
- **Notifications**: set up SMTP and `PORTAL_ENCRYPTION_KEY`. For Microsoft 365, prefer an app-specific relay or OAuth-capable connector over a personal mailbox password. Teams delivery could be added in `core.services.notify`.
- **Data**: move to PostgreSQL. Store uploads in managed storage (SharePoint / S3) with virus scanning.
- **Business rules**: SLA uses 24x7 time (add a business-hours / public-holiday calendar if needed). Claim totals are summed across currencies (add FX conversion if needed).
- **Review**: this is AI-generated code. Finance, IT Security and Internal Audit should review the approval rules and access controls before real use.
