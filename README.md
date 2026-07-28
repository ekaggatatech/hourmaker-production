# HourMaker

Internal workforce and HR operations platform for **Ekaggata** (product UI name: **HourMaker**).  
It covers time tracking, leave, projects, recruitment, referrals, onboarding, helpdesk, documents, billing, and admin setup — with role-based access.

## Full documentation (presentation-ready)

See **[docs/HourMaker-Project-Documentation.md](docs/HourMaker-Project-Documentation.md)** for:

- Complete feature catalog  
- End-to-end process flows (Mermaid)  
- Entity-relationship diagram (ERD)  
- Data dictionary, Cloud Functions, security model  
- Demo / presentation talking points  

---

## Tech stack

| Layer | Technology |
|--------|------------|
| Frontend | React 19, React Router, Tailwind / MUI / Radix |
| Backend | Firebase Auth, Firestore, Storage, Cloud Functions (Node 20) |
| Email | Firestore `mail` collection (Trigger Email extension) |
| Charts / PDF | Recharts, Chart.js, jsPDF, `@react-pdf/renderer`, xlsx |
| Local app | Create React App (`react-scripts`) |

---

## Getting started

```bash
# App
npm install
npm start          # http://localhost:3000

# Cloud Functions
cd functions
npm install
npm run deploy     # or: npm run serve for emulators
```

Firebase client config lives under `src/Components/firebaseConfig.js`. You need a Firebase project with Auth, Firestore, Storage, and Functions enabled for full features.

---

## Roles and entry points

Login redirects by `users.role`:

| Role | Home path |
|------|-----------|
| `admin` | `/admin/:userId` |
| `subadmin` | `/subadmin/:userId` |
| `hr manager` / `hr executive` | `/hr/:userId` |
| `manager` | `/manager/:userId` |
| `employee` | `/employee/:userId` |
| `pr employee` | `/pr-employee/:userId` |

Fine-grained menu/actions also use Firestore `role_permissions/{role}`.

### What each role typically uses

- **Employee** — Dashboard, timesheet / time tracker, leave (via flows), meetings, tasks, reports, Docs, Referrals, Helpdesk  
- **Manager** — Team projects, leave & timesheet approvals, users/teams, Docs, referrals, announcements/meetings  
- **Admin** — Full setup + approvals + Recruitment + Onboarding + Referrals + Billing/Invoices + Docs + Helpdesk  
- **Subadmin** — Subset of admin (projects, approvals, setup, Docs); no full recruitment/onboarding/invoice surface in sidebar  
- **HR Manager** — Recruitment (full), referrals, onboarding first review, leave/timesheet approvals, Docs, reports  
- **HR Executive** — Assigned recruitment work, Drive, referrals as assigned, onboarding view (no first-approve unless manager)  
- **PR Employee** — Reference form + Onboarding form only  

---

## Feature modules

### 1. Timesheets and time tracking

- Weekly timesheet create / submit / resubmit  
- Manager / Admin / HR / Subadmin approval  
- Separate **Time Tracker** (entries, breaks, idle) that can feed timesheets  
- Admin **Timesheet Settings**; Friday reminders via scheduled Cloud Functions  
- Monthly hour aggregates (`userMonthlyHours`, team activity)

### 2. Leave

- Employee leave requests  
- Approvals by manager / admin / HR  
- Leave types and balances (`leaves`, `user_leaves`)  
- Monthly earned leave credit and year-end carry-forward (scheduled)

### 3. Projects, clients, teams, users

- Clients and projects (contributors, managers)  
- Teams and user management (create/update/delete via Cloud Functions)  
- Roles and **Access Management** (`role_permissions`)  
- Shifts and company holiday calendar  

### 4. Recruitment Tracker

Located under Recruitment for Admin / HR.

| Tab | Purpose |
|-----|---------|
| Dashboard | KPIs and charts from live requirements + activity |
| Requirement Tracker | List/filter/paginate jobs; open details; daily activity logs |
| New Requirement | Create Internal or Client Side jobs (assign HR) |
| Drive | Hiring drives with deadline reminders |
| Reports | Export Excel/PDF |

**Notable behaviors**

- **Aging:** Calendar days since assignment (Sundays excluded); orange after 5 days, red after 10; email + in-app via `recruitmentAgingReminder`  
- **Requirement Information:** Admin/Subadmin can edit core fields (e.g. open positions); HR can update status and log activity  
- **Drive:** Fields include drive date, job id, skill, positions, SPOCs, deadline, totals, notify recipients (name search → email from `users`); daily reminders via `recruitmentDriveDeadlineReminder`; Admin can **Run reminders now**  
- Collections: `recruitmentRequirements`, `recruitmentSummaries`, `recruitmentActivityLogs`, `recruitmentDrives`

### 5. Referrals and job openings

- Employees / managers / HR / admin submit referrals against `jobOpenings`  
- Statuses: Pending Assignment → Screening → Interview → Final Discussion → On Hold → Hired / Rejected  
- Assign / self-assign (HR Manager); status updates via Cloud Functions  
- Lists support search, status filter, pagination (10 per page)

### 6. Onboarding (two-step approval)

PR employees submit **Reference** then **Onboarding** forms (`referenceForms`, `employeeForms`).

```text
Employee submit
    → Pending HR
        → HR Manager Approve → Pending Admin
        → HR Manager Reject  → Rejected → employee edits & resubmits → Pending HR
            → Admin Approve → Approved (user promoted to employee)
            → Admin Reject  → Rejected → employee resubmits → Pending HR
```

- HR list defaults to **Pending HR**; Admin list defaults to **Pending Admin**  
- Duplicate submissions after reject are upserted (one active form per candidate; list deduped)  
- Cloud Functions: `onOnboardingApproved`, `onOnboardingRejected`

### 7. Helpdesk

- Tickets in `helpdesk_requests` with categories  
- Create / resolve notifications and emails via Cloud Functions  

### 8. Docs Portal

- Folders and files in Firestore + Storage  
- Share / search users by name or email  

### 9. Meetings, announcements, reports

- Schedule meetings and send announcements (notifications)  
- Personal, weekly, manager, and company reports  

### 10. Billing and invoices

- Price / billing dashboard from projects and hours  
- Manual and auto invoice flows; PDF view; `generateInvoice` callable  

### 11. Employment lifecycle

- Contract expiry reminders (`contractExpiryReminder`)  
- Auto-deactivate expired contractual / project trainee users (`deactivateExpiredEmployees`)  

---

## High-level architecture

```text
React SPA (role layouts + sidebars)
        │
        ▼
Firebase Auth + Firestore (live data / rules)
        │
        ├── Storage (CVs, onboarding files, docs)
        ├── Cloud Functions (callable + scheduled + triggers)
        └── mail collection → email delivery
```

Notifications land in `notifications` and show in the header bell for the signed-in `userId`.

---

## Important Firestore collections

| Area | Collections |
|------|-------------|
| Identity | `users`, `roles`, `role_permissions` |
| Work | `timesheets`, `projects`, `clients`, `teams`, `activities`, `shifts` |
| Leave | `leaveRequests`, `leaves`, `user_leaves`, `holidays` |
| Recruitment | `recruitmentRequirements`, `recruitmentSummaries`, `recruitmentActivityLogs`, `recruitmentDrives` |
| Referrals | `referrals`, `jobOpenings` |
| Onboarding | `employeeForms`, `referenceForms` |
| Comms | `notifications`, `meetings`, `announcements`, `mail` |
| Docs | `files`, `folders` |
| Billing | `invoices`, `userMonthlyHours`, `settings` |
| Helpdesk | `helpdesk_requests`, `categories` |

---

## Notable Cloud Functions

### Scheduled (IST unless noted)

| Function | When | Purpose |
|----------|------|---------|
| `sendReminder` | Fri 17:25 | Timesheet email reminders |
| `recruitmentAgingReminder` | Daily 09:30 | Job aging orange/red alerts |
| `recruitmentDriveDeadlineReminder` | Daily 09:30 | Drive deadline reminders |
| `contractExpiryReminder` | Daily 09:00 | Contract end alerts |
| `deactivateExpiredEmployees` | Daily 00:05 | Deactivate expired contracts |
| `addMonthlyEarnedLeave` | 1st of month | Earned leave credit |
| `yearEndCarryForward` | Jan 1 | Leave carry-forward |
| `calculateMonthlyHours` | Fri 01:00 | Monthly hours rollup |

### Triggers

- `onOnboardingApproved` / `onOnboardingRejected` on `employeeForms`  
- `onTicketCreated` / `onTicketResolved` on helpdesk  
- `handleLeaveApproval` on leave requests  
- `trackTimesheetWeek` on timesheets  

### Useful callables

User CRUD, timesheet emails, referrals (`submitReferral`, `updateReferralStatus`, `assignReferral`), `getSignedFileUrl`, `triggerRecruitmentDriveDeadlineReminder` (Admin force-send), invoice generation, client/project helpers.

---

## Source layout (high level)

```text
src/
  App.js                 # Routes by role
  AuthContext.js         # Auth + role + permissions
  Components/
    Admin/               # Admin dashboard, users, onboarding, billing, …
    HR/                  # HR dashboard, onboarding review, …
    Manager/             # Manager flows
    EmployeeDashboard/   # Employee home
    Recruitment/         # Tracker, Drive, Reports, aging
    ReferralManagement/  # Referrals + job openings
    PrDashboard/         # Reference + onboarding forms
    docsComponent/       # Docs portal
    …                    # Timesheets, leave, helpdesk, etc.
functions/
  index.js               # All Cloud Functions
firestore.rules          # Security rules
```

---

## Security notes

- Access is enforced by **Firestore rules** (role helpers like `isAdmin`, `isHR`, `isHRManager`) and **UI role checks**.  
- Sensitive writes (user create/delete, some emails) go through **Cloud Functions** with Admin SDK.  
- Recruitment drives and requirements have field-level update limits for HR vs Admin where configured.

---

## Related docs

- Create React App scripts: original CRA notes may still exist under `docs/`  
- Recruitment Firestore setup notes: `src/Components/Recruitment/FIRESTORE_SETUP.md` (if present)

---

*This overview reflects the application as implemented in the repository. Exact menu items can also be toggled per role via Access Management.*
