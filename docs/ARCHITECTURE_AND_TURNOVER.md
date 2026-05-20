# Overtime System — Architecture & Turnover Documentation

> **Version:** 1.0 | **Date:** 2026-05-18 | **System:** Overtime Request Management System

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Technology Stack](#2-technology-stack)
3. [System Architecture](#3-system-architecture)
4. [Database Architecture](#4-database-architecture)
5. [Role-Based Access Guide](#5-role-based-access-guide)
6. [Process Flows](#6-process-flows)
   - [Authentication Flow](#61-authentication-flow)
   - [OT Request Creation Flow](#62-ot-request-creation-flow)
   - [Approval Workflow Flow](#63-approval-workflow-flow)
   - [Bulk Approval Flow](#64-bulk-approval-flow)
   - [Cancellation Flow](#65-cancellation-flow)
   - [Export Flow](#66-export-flow)
   - [Dashboard Aggregation Flow](#67-dashboard-aggregation-flow)
   - [Operator Management Flow](#68-operator-management-flow)
7. [External Integrations](#7-external-integrations)
8. [Status Reference](#8-status-reference)
9. [Infrastructure & Configuration](#9-infrastructure--configuration)
10. [Codebase Map](#10-codebase-map)
11. [Known Gotchas & Notes](#11-known-gotchas--notes)

---

## 1. System Overview

The **Overtime System** is an internal HR web application that manages employee overtime (OT) requests through a multi-level approval workflow. It integrates with the company's HRIS, Payroll, and SSO (Authify) services.

### Core Capabilities

| Capability | Description |
|---|---|
| OT Request Submission | Employees create OT requests for themselves and their direct reports |
| Multi-Level Approval | Up to 3 approval tiers resolved from the HRIS hierarchy |
| Bulk Actions | Approvers can approve or disapprove multiple requests at once |
| Analytics Dashboard | Summary stats, trends, department breakdowns by role scope |
| Operator Admin Panel | Admins manage operators and export OT data to CSV |
| SSO Authentication | Single Sign-On via Authify — no separate login credentials |

---

## 2. Technology Stack

### Backend
| Layer | Technology |
|---|---|
| Framework | Laravel 12 (PHP 8.2+) |
| Frontend Bridge | Inertia.js 2.0 (React adapter) |
| Authentication | Sanctum 4.0 + Authify SSO |
| ORM | Eloquent |
| Queue/Cache | Database-backed |
| Session | File-based (12-hour lifetime) |

### Frontend
| Layer | Technology |
|---|---|
| UI Framework | React 18 |
| Styling | Tailwind CSS 3 + DaisyUI 5 |
| Components | ShadCN/UI (Radix UI) + Headless UI |
| Forms | React Hook Form |
| State | Zustand 5 |
| Charts | Chart.js + react-chartjs-2 |
| Build Tool | Vite 6 |

### Infrastructure
| Component | Detail |
|---|---|
| Primary DB | MySQL — `overtime` database |
| Employee DB | MySQL — `masterlist` database |
| Auth DB | MySQL — `authify` database |
| App Server | PHP 8.2+ / Laravel Sail (Docker) |
| Reverse Proxy | Nginx |
| Docker Service Name | `medical` |
| Exposed Port | `8313` |

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         BROWSER (React)                         │
│  Dashboard │ OT Requests │ Operators │ Admin │ Profile          │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HTTPS (Inertia.js)
┌─────────────────────────▼───────────────────────────────────────┐
│                     NGINX (Reverse Proxy)                       │
│                    Port 8313 → Laravel                          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                  LARAVEL APPLICATION                            │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐    │
│  │ Middleware   │  │ Controllers │  │     Services         │    │
│  │ AuthMiddle  │→ │ Overtime    │→ │ OvertimeService      │    │
│  │ AdminMiddle │  │ Dashboard   │  │ DashboardService     │    │
│  └─────────────┘  │ OtOperator  │  │ HrisApiService       │    │
│                   │ Admin       │  │ PayrollApiService    │    │
│                   │ Profile     │  │ OtOperatorService    │    │
│                   └─────────────┘  └──────────┬──────────┘    │
│                                               │                │
│                   ┌───────────────────────────▼──────────┐    │
│                   │           Repositories                │    │
│                   │  OvertimeRepository                   │    │
│                   │  DashboardRepository                  │    │
│                   │  OtOperatorRepository                 │    │
│                   └───────────────────────────────────────┘    │
└──────────────┬──────────────────────────┬──────────────────────┘
               │                          │
    ┌──────────▼──────────┐   ┌───────────▼──────────────────┐
    │   MySQL Databases   │   │     External APIs            │
    │                     │   │                              │
    │  [overtime]         │   │  HRIS :8000                  │
    │  ot_requests_new    │   │  - employees                 │
    │  ot_items_new       │   │  - work details              │
    │  ot_approvals       │   │  - direct reports            │
    │  ot_approver_roles  │   │  - bulk employee info        │
    │  ot_operators       │   │                              │
    │  admin              │   │  Payroll :8004               │
    │  system_status      │   │  - payroll cutoff schedules  │
    │                     │   │                              │
    │  [masterlist]       │   │  Authify :8001 (SSO)         │
    │  employee_masterlist│   │  - token validation          │
    │                     │   │  - session management        │
    │  [authify]          │   │  - logout redirect           │
    │  authify_sessions   │   │                              │
    └─────────────────────┘   └──────────────────────────────┘
```

---

## 4. Database Architecture

### 4.1 Database Connections

| Connection Key | Database Name | Purpose |
|---|---|---|
| `mysql` (default) | `overtime` | All OT application data |
| `masterlist` | `masterlist` | Employee master list lookup |
| `authify` | `authify` | SSO session token validation |

### 4.2 Core Tables (`overtime` database)

#### `ot_requests_new` — Overtime Request Header
| Column | Type | Description |
|---|---|---|
| `ot_form_no` | varchar (PK) | Auto-generated: `OTyy-XXXX` format |
| `requestor_id` | varchar | Employee ID of requestor |
| `department` | varchar | Department name |
| `productline` | varchar | Product line / station |
| `date_from` | date | OT period start |
| `date_to` | date | OT period end |
| `ot_status` | tinyint | Request status (see Status Reference) |
| `created_at` | timestamp | Submission timestamp |

#### `ot_items_new` — Overtime Request Line Items
| Column | Type | Description |
|---|---|---|
| `id` | bigint (PK) | Auto-increment |
| `ot_form_code` | varchar (FK) | References `ot_requests_new.ot_form_no` |
| `emp_num` | varchar | Employee number on OT |
| `work_shift` | varchar | Employee's work shift |
| `ot_date_from` | datetime | OT start datetime |
| `ot_time_from` | time | OT start time |
| `ot_time_to` | time | OT end time |
| `ot_reason` | text | Reason for overtime |

#### `ot_approvals` — Multi-Level Approval Records
| Column | Type | Description |
|---|---|---|
| `id` | bigint (PK) | Auto-increment |
| `ot_form_no` | varchar (FK) | References `ot_requests_new.ot_form_no` |
| `role_id` | tinyint (FK) | References `ot_approver_roles.id` (1/2/3) |
| `approver_name` | varchar | Employee ID of approver (set after action) |
| `status` | tinyint | Approval status (0=pending, 1=approved, 2=rejected) |
| `remarks` | text | Approver remarks |
| `signed_at` | timestamp | When action was taken |

#### `ot_approver_roles` — Approval Role Definitions
| Column | Type | Description |
|---|---|---|
| `id` | tinyint (PK) | 1, 2, or 3 |
| `role_code` | varchar | Short code |
| `role_name` | varchar | Display name |

#### `ot_operators` — System Operators/Admins
| Column | Type | Description |
|---|---|---|
| `ot_emp_num` | varchar (PK) | Employee number |
| `ot_emp_name` | varchar | Employee name |
| `ot_access_level` | tinyint | 1=normal, 2=admin |
| `ot_account_status` | tinyint | 1=active, 2=inactive |
| `date_created` | timestamp | Creation date |
| `date_updated` | timestamp | Last update date |

#### `admin` — Admin Users
| Column | Type | Description |
|---|---|---|
| `emp_id` | varchar (PK) | Employee ID |
| `emp_name` | varchar | Employee name |
| `emp_role` | varchar | Admin role |
| `last_updated_by` | varchar | Who last modified |

#### `system_status` — Maintenance Mode Toggle
| Column | Type | Description |
|---|---|---|
| `status` | tinyint | 1=online, 0=maintenance |
| `message` | text | Maintenance message shown to users |
| `updated_at` | timestamp | Last toggle time |

### 4.3 Entity Relationships

```
ot_requests_new (1) ──────── (many) ot_items_new
      │                              [via ot_form_no]
      │
      └── (1) ──────── (many) ot_approvals
                               [via ot_form_no]
                                    │
                                    └── (many) ──── (1) ot_approver_roles
                                                        [via role_id]
```

---

## 5. Role-Based Access Guide

### 5.1 Role Determination

The system does **not** have a roles table. Roles are resolved at runtime based on:

```
Is the user in ot_operators (status=1)?
  YES → OPERATOR role
  NO  →
    Does the user have direct reports in HRIS?
    OR is emp_position_id = 6 (Operations Director)?
      YES → APPROVER role
      NO  → EMPLOYEE role
```

### 5.2 Role Capabilities Matrix

| Feature | Employee | Approver | Operator |
|---|---|---|---|
| View own OT requests | ✅ | ✅ | ✅ |
| Create OT request (self) | ✅ | ✅ | ✅ |
| Create OT request (for direct reports) | ❌ | ✅ | ✅ |
| Approve/Disapprove requests | ❌ | ✅ (team only) | ❌ |
| Bulk Approve/Disapprove | ❌ | ✅ (team only) | ❌ |
| View team requests | ❌ | ✅ | ✅ (all) |
| Dashboard — own scope | ✅ | ✅ (team) | ✅ (all) |
| Export to CSV | ❌ | ❌ | ✅ |
| Manage Operators | ❌ | ❌ | ✅ |
| Admin Panel access | ❌ | ❌ | ✅ (if in admin table) |
| Cancel own requests | ✅ | ✅ | ✅ |

### 5.3 Role-Specific Dashboard Scope

| Role | What They See in Dashboard |
|---|---|
| Employee | Only their own submitted requests |
| Approver | Their team's requests (direct reports) |
| Operator | All requests across all departments |
| Operations Director (pos. 6) | All requests |

### 5.4 Special Case: Operations Director (Position 6)

- Identified by `emp_position_id = 6` in the SSO session
- Can approve requests that are already **Partially Approved** (bypasses normal hierarchy)
- Treated as an approver with full visibility across departments

---

## 6. Process Flows

### 6.1 Authentication Flow

```
User opens browser
       │
       ▼
AuthMiddleware checks for token
  (priority: ?key= param → cookie [sso_token] → session [emp_data.token])
       │
       ├── Token NOT found
       │      └── Redirect to Authify login (:8001)
       │
       └── Token found → validate against authify.authify_sessions
              │
              ├── Invalid/expired token
              │      └── Redirect to Authify login
              │
              └── Valid token
                     │
                     ├── Check system_status → if maintenance: show maintenance page
                     │
                     └── Populate session:
                            emp_id, emp_name, emp_firstname,
                            emp_dept_id, emp_jobtitle_id,
                            emp_position_id, shift_type, team
                            └── Proceed to requested route
```

**Logout:**
```
User clicks Logout
    → GET /Overtime/logout
    → Clear session
    → Redirect to Authify /logout?token={token}&redirect={app_url}
    → Authify invalidates token
    → Redirect back to login page
```

---

### 6.2 OT Request Creation Flow

**Actors:** Employee, Approver (Manager), Operator

```
1. GET /Overtime/create
        │
        ├── Fetch requestor work details from HRIS
        │     (department, productline, shift_type, approver hierarchy)
        │
        ├── Fetch list of direct reports + self from HRIS
        │
        ├── Fetch payroll cutoff schedules from Payroll API
        │
        └── Render Create Form

2. User fills form:
        │
        ├── Department (auto-filled from HRIS)
        ├── Product Line (auto-filled)
        ├── Date From / Date To (select from payroll cutoff periods)
        ├── Employee list (self + direct reports selectable)
        │     For each employee:
        │       - OT Date From/To
        │       - OT Time From/To
        │       - Work Shift
        │       - Reason
        └── Submit

3. POST /Overtime/
        │
        ├── Validate:
        │     - department (required)
        │     - productline (required)
        │     - date_from / date_to (required, valid dates)
        │     - employees[] array (required, at least 1)
        │     - each employee: emp_num, ot_date_from, ot_date_to, ot_reason
        │
        ├── DB Transaction:
        │     ├── Generate unique form_no: OTyy-XXXX
        │     │     (padded sequential number per year)
        │     ├── INSERT ot_requests_new (status = 0 = Pending)
        │     ├── INSERT ot_items_new (one row per employee)
        │     └── INSERT ot_approvals:
        │           - role_id 1 (Approver Level 1) → status = 0 (pending)
        │           - role_id 2 (Approver Level 2) → status = 0 (pending)
        │           - role_id 3 (Approver Level 3) → status = 0 (pending)
        │
        └── Redirect → GET /Overtime/{formNo}
```

---

### 6.3 Approval Workflow Flow

**Actors:** Approver (Manager), Operations Director

```
1. GET /Overtime/{formNo}
        │
        ├── Load OvertimeRequest with items and approvals
        ├── Bulk fetch employee names from HRIS (by emp_nums)
        ├── Resolve which approval record belongs to current user:
        │     Check 1: Is requestor's HRIS approver1/2/3 == current user?
        │     Check 2: Is current user a direct manager of requestor?
        │     Check 3: Is current user emp_position_id=6 AND request is Partially Approved?
        ├── If user has a matching approval record → show Approve/Disapprove buttons
        └── Render request detail + approval section

2. POST /Overtime/{formNo}/approve  OR  /disapprove
        │
        ├── Validate remarks (required, max 500 characters)
        │
        ├── Re-resolve approval record (same 3 checks as above)
        │     └── If not authorized → abort 403
        │
        ├── DB Transaction:
        │     ├── UPDATE ot_approvals:
        │     │     status    → 1 (approved) or 2 (rejected)
        │     │     approver_name → emp_id of current user
        │     │     remarks   → user input
        │     │     signed_at → now()
        │     │
        │     └── Recalculate ot_requests_new.ot_status:
        │           ┌─ Any approval.status = 2 (rejected)?
        │           │     → ot_status = 3 (Disapproved)
        │           ├─ All approvals.status = 1 (approved)?
        │           │     → ot_status = 2 (Approved)
        │           ├─ Some approvals.status = 1, rest = 0?
        │           │     → ot_status = 1 (Partially Approved)
        │           └─ All still pending?
        │                 → ot_status = 0 (Pending)
        │
        └── Redirect back with flash success message
```

**Approval Hierarchy Visualization:**
```
                       Request Submitted
                             │
                    [Level 1 Approver]
                    (HRIS approver1)
                      pending → approved
                             │
                    [Level 2 Approver]
                    (HRIS approver2)
                      pending → approved
                             │
                    [Level 3 Approver]
                    (HRIS approver3)
                      pending → approved
                             │
                         APPROVED ✅

  At any level: if REJECTED → entire request = Disapproved ❌
  Operations Director (pos.6): can act on Partially Approved requests
```

---

### 6.4 Bulk Approval Flow

**Actors:** Approver (Manager)

```
POST /Overtime/bulk-action
        │
        ├── Validate:
        │     - form_nos: array of form numbers (required)
        │     - action: "approve" or "disapprove" (required)
        │     - remarks: string (required, max 500)
        │
        ├── Initialize counters: success=0, failed=0
        │
        ├── For each form_no in form_nos:
        │     ├── Find OvertimeRequest
        │     ├── Resolve approval record for current user
        │     ├── Call processApproval (same logic as single approval)
        │     ├── On success → success++
        │     └── On error → log warning, failed++
        │
        └── Return response:
              "{success} request(s) processed. {failed} failed."
```

---

### 6.5 Cancellation Flow

**Actors:** Employee (requestor only)

```
PATCH /Overtime/{formNo}/cancel
        │
        ├── Verify current user == requestor_id
        │     └── If not → abort 403
        │
        ├── Verify ot_status == 0 (Pending)
        │     └── If not Pending → abort 422 (cannot cancel non-pending)
        │
        ├── UPDATE ot_requests_new SET ot_status = 4 (Cancelled)
        │
        └── Redirect back with flash message
```

---

### 6.6 Export Flow

**Actors:** Operator only

```
GET /Overtime/export?status=&date_from=&date_to=
        │
        ├── Verify current user is Operator
        │     └── If not → abort 403
        │
        ├── Apply filters:
        │     - status (optional: 0/1/2/3/4)
        │     - date_from / date_to (optional date range)
        │
        ├── Query ot_requests_new with ot_items_new joined
        │
        ├── Build CSV content with UTF-8 BOM (for Excel compatibility)
        │     Columns:
        │       EMP NO | NAME | DATE REQUESTED | TIME START
        │       TIME END | HOURS | REMARKS | DATE APPROVED | STATUS
        │
        └── Return CSV download response
              filename: overtime_export_{timestamp}.csv
```

---

### 6.7 Dashboard Aggregation Flow

**Actors:** All roles (scoped differently)

```
GET /Overtime (Dashboard)
        │
        ├── Determine role and resolve scope:
        │     ├── Operator or pos.6 → scope = null (all data)
        │     ├── Approver → scope = [list of direct report emp_ids]
        │     └── Employee → scope = [own emp_id]
        │
        ├── Fetch stat cards:
        │     - Total requests (in scope)
        │     - Pending count
        │     - Partially Approved count
        │     - Approved count
        │     - Disapproved count
        │     - Cancelled count
        │     - Distinct employees on OT (from ot_items_new)
        │
        ├── Fetch monthly trend (12 months):
        │     GROUP BY month for selected year (default: current year)
        │
        ├── Fetch top 10 departments by request count
        │
        ├── Fetch status distribution (for pie chart)
        │
        ├── Fetch 10 most recent requests
        │
        └── Render Dashboard with Chart.js visualizations:
              - Bar chart (monthly trend)
              - Pie chart (status distribution)
              - Table (top departments)
              - Stat cards
              - Recent requests list
```

---

### 6.8 Operator Management Flow

**Actors:** Operator (Admin-level)

```
LIST:   GET /Overtime/operators
        └── Paginated list with search by emp_num / emp_name

CREATE: GET /Overtime/operators/employees/search?q={query}
        ├── Calls HRIS /api/employees/active with search param
        └── Returns employee options for combobox

        POST /Overtime/operators
        ├── Validate: emp_num (unique), ot_access_level (1 or 2), ot_account_status (1 or 2)
        ├── INSERT ot_operators
        └── Redirect with success

UPDATE: PUT /Overtime/operators/{operator}
        ├── Validate same fields
        ├── UPDATE ot_operators SET ... WHERE ot_emp_num = {operator}
        └── Redirect with success

DELETE: DELETE /Overtime/operators/{operator}
        ├── DELETE FROM ot_operators WHERE ot_emp_num = {operator}
        └── Redirect with success
```

---

## 7. External Integrations

### 7.1 HRIS API (`http://127.0.0.1:8000`)

**Auth Header:** `X-Internal-Key: {HRIS_API_KEY}`

| Endpoint | Purpose | Used By |
|---|---|---|
| `GET /api/employees/{id}` | Get employee name/info | Show page |
| `GET /api/employees/{id}/work` | Get dept, productline, shift, approver hierarchy | Create form |
| `GET /api/employees/{id}/salary` | Salary data | Profile |
| `GET /api/employees/operation-director` | Get position-6 employee | Approval resolver |
| `GET /api/employees/active?search=&page=` | Paginated active employee search | Operator creation |
| `GET /api/employees/direct-reports/{id}` | List direct reports | Create form, Approver check |
| `POST /api/employees/bulk` body: `{emp_nos:[]}` | Batch fetch employee names | Show page |

### 7.2 Payroll API (`http://127.0.0.1:8004`)

**Auth Header:** `X-Internal-Key: {PAYROLL_API_KEY}`

| Endpoint | Purpose | Used By |
|---|---|---|
| `GET /api/payroll-cutoff-schedules?year=` | Get cutoff periods | Create form (date picker) |
| `GET /api/payroll-cutoff-schedules/{id}` | Single cutoff record | Reference |

### 7.3 Authify SSO (`http://127.0.0.1:8001`)

**Integration type:** Direct database read + HTTP redirect

| Action | Method |
|---|---|
| Token validation | Query `authify.authify_sessions` table directly |
| Logout | Redirect to `/logout?token={token}&redirect={url}` |

---

## 8. Status Reference

### OT Request Status (`ot_requests_new.ot_status`)

| Value | Label | Description |
|---|---|---|
| `0` | Pending | Newly submitted, no approvals yet |
| `1` | Partially Approved | At least one level approved, not all |
| `2` | Approved | All approval levels approved |
| `3` | Disapproved | At least one level rejected |
| `4` | Cancelled | Cancelled by requestor (only from Pending) |

### OT Approval Status (`ot_approvals.status`)

| Value | Label | Description |
|---|---|---|
| `0` | Pending | Awaiting approver action |
| `1` | Approved | Approver approved |
| `2` | Rejected | Approver rejected |

### Operator Account Status (`ot_operators.ot_account_status`)

| Value | Label |
|---|---|
| `1` | Active |
| `2` | Inactive |

### Operator Access Level (`ot_operators.ot_access_level`)

| Value | Label |
|---|---|
| `1` | Normal |
| `2` | Admin |

---

## 9. Infrastructure & Configuration

### 9.1 Environment Variables (`.env`)

| Variable | Example Value | Purpose |
|---|---|---|
| `APP_NAME` | `Overtime` | Application name |
| `APP_TIMEZONE` | `Asia/Manila` | Timezone for timestamps |
| `APP_KEY` | generated | Laravel encryption key |
| `DB_CONNECTION` | `mysql` | Default DB connection |
| `DB_HOST` | `127.0.0.1` | Primary DB host |
| `DB_PORT` | `3306` | Primary DB port |
| `DB_DATABASE` | `overtime` | Primary DB name |
| `DB_USERNAME` | `root` | Primary DB user |
| `DB_PASSWORD` | — | Primary DB password |
| `MDB_DATABASE` | `masterlist` | Masterlist DB name |
| `ADB_DATABASE` | `authify` | Authify DB name |
| `SSO_COOKIE_NAME` | `sso_token` | Cookie key for SSO token |
| `HRIS_API_URL` | `http://127.0.0.1:8000` | HRIS service URL |
| `HRIS_API_KEY` | `hris-secret-123456` | HRIS auth key |
| `PAYROLL_API_URL` | `http://127.0.0.1:8004` | Payroll service URL |
| `PAYROLL_API_KEY` | `ws-secret-123456` | Payroll auth key |
| `SESSION_DRIVER` | `file` | Session storage |
| `SESSION_LIFETIME` | `720` | Session TTL (minutes = 12h) |
| `QUEUE_CONNECTION` | `database` | Queue driver |
| `CACHE_STORE` | `database` | Cache driver |

### 9.2 Docker Commands

```bash
# Build and start the application
cd /var/www
docker compose up -d --build --no-deps medical
docker compose up -d --no-deps nginx

# Post-deployment setup
docker compose exec medical php artisan key:generate
docker compose exec medical php artisan migrate --force
docker compose exec medical php artisan storage:link

# Cache for production
docker compose exec medical php artisan config:cache
docker compose exec medical php artisan route:cache
docker compose exec medical php artisan view:cache

# Clear cache (if needed)
docker compose exec medical php artisan route:clear
docker compose exec medical php artisan config:clear

# Verify the app is responding
curl -v --max-redirs 0 http://192.168.1.12:8313/ 2>&1 | grep "Location"
```

### 9.3 Routes Files

| File | Prefix | Contents |
|---|---|---|
| `routes/auth.php` | `/Overtime` | Logout |
| `routes/general.php` | `/Overtime` | Dashboard, profile, admin panel |
| `routes/overtime.php` | `/Overtime` | OT request CRUD, operators |
| `routes/web.php` | `/` | Root redirect |

---

## 10. Codebase Map

### Backend (`app/`)

```
app/
├── Http/
│   ├── Controllers/
│   │   ├── AuthenticationController.php      # Logout
│   │   ├── DashboardController.php           # Dashboard data
│   │   ├── OvertimeController.php            # OT CRUD + approvals (main controller)
│   │   ├── OtOperatorController.php          # Operator management
│   │   └── General/
│   │       ├── AdminController.php           # Admin user management
│   │       └── ProfileController.php         # User profile / password
│   ├── Middleware/
│   │   ├── AuthMiddleware.php                # SSO token validation, session hydration
│   │   └── AdminMiddleware.php               # Admin-only route guard
│   └── Requests/                             # Form validation request classes
│
├── Models/
│   ├── OvertimeRequest.php                   # ot_requests_new
│   ├── OvertimeItem.php                      # ot_items_new
│   ├── OtApproval.php                        # ot_approvals
│   ├── OtApproverRole.php                    # ot_approver_roles
│   ├── OtOperator.php                        # ot_operators
│   ├── SystemStatus.php                      # system_status
│   └── User.php                              # users (fallback)
│
├── Services/
│   ├── OvertimeService.php                   # Create / approve / bulk logic
│   ├── OtOperatorService.php                 # Operator CRUD logic
│   ├── DashboardService.php                  # Stat aggregation + scope resolution
│   ├── HrisApiService.php                    # All HRIS API calls
│   ├── PayrollApiService.php                 # Payroll cutoff API calls
│   ├── SystemStatusService.php               # Maintenance mode check
│   └── DataTableService.php                  # Generic paginate/search helper
│
└── Repositories/
    ├── OvertimeRepository.php                # OT query builders (filters, pagination)
    ├── DashboardRepository.php               # Dashboard aggregation queries
    └── OtOperatorRepository.php              # Operator queries
```

### Frontend (`resources/js/`)

```
resources/js/
├── Pages/
│   ├── Dashboard.jsx                         # Main dashboard with charts
│   ├── Profile.jsx                           # User profile + change password
│   ├── Unauthorized.jsx                      # 403 page
│   ├── 404.jsx                               # Not found page
│   ├── Authentication/
│   │   └── Login.jsx                         # SSO redirect page
│   ├── Overtime/
│   │   ├── Index.jsx                         # OT request list (pending/history/approvals)
│   │   ├── Create.jsx                        # Create new OT request form
│   │   ├── Show.jsx                          # Request detail + approval action
│   │   ├── Operators/
│   │   │   └── Index.jsx                     # Operator management CRUD
│   │   └── components/
│   │       ├── ApprovalBlock.jsx             # Approve/disapprove UI block
│   │       ├── ConfirmCancelDialog.jsx       # Cancel confirmation dialog
│   │       ├── ExportDialog.jsx              # Export filter + download dialog
│   │       ├── FilterBar.jsx                 # Table filter controls
│   │       ├── RemarksDialog.jsx             # Remarks input modal
│   │       ├── RowActions.jsx                # Table row action buttons
│   │       └── StatusBadge.jsx              # Colored status label
│   └── Admin/
│       ├── Admin.jsx                         # Admin management table
│       └── NewAdmin.jsx                      # Add new admin form
│
├── Components/
│   ├── ui/                                   # Reusable primitives (buttons, forms, dialogs, tables)
│   └── sidebar/                              # Navigation sidebar
│
└── helpers/                                  # Utility/helper functions
```

---

## 11. Known Gotchas & Notes

### Form Number Generation
- Format: `OTyy-XXXX` where `yy` = 2-digit year, `XXXX` = zero-padded sequential count per year
- Generated inside a DB transaction — safe from race conditions

### Approval Resolution Logic
The system resolves which approval record belongs to the current user using 3 sequential checks. The order matters:
1. HRIS hierarchy match (approver1/2/3 field = current user's emp_id)
2. Direct reports check (requestor is a direct report of current user)
3. Position-6 override (only works on Partially Approved requests)

If none match, the user cannot approve — no buttons shown.

### Route Caching in Production
Routes in `routes/*.php` must NOT use `env()` calls directly. Use `config()` instead. Violation causes `route:cache` to fail. See `Deploy 12.md` for the `sed` fix commands if needed.

### Multi-Database Queries
HRIS employee data and masterlist data are never stored locally. They are always fetched live from the HRIS API. This means network failures to `:8000` will break the Create form and Show page bulk-name fetch.

### SSO Cookie Name
The cookie name is configurable via `SSO_COOKIE_NAME` env var (default: `sso_token`). If changed in Authify, it must also be updated in `.env` here.

### Session Lifetime
Sessions expire after 720 minutes (12 hours). Users are silently logged out — no warning before expiry. The SSO token in the cookie must still be valid for re-authentication to work without prompting for a login screen.

### Bulk Action Failure Handling
Bulk actions are **not atomic**. If 3 out of 5 requests succeed and 2 fail, the 3 are committed. Failures are logged as warnings. The response message reports counts but does not identify which form numbers failed.

### Maintenance Mode
Controlled via the `system_status` table. When `status = 0`, all authenticated users (except those who bypass it — check `AuthMiddleware.php`) are shown a maintenance page. This is separate from Laravel's built-in `php artisan down`.

---

*End of Overtime System Architecture & Turnover Documentation*
