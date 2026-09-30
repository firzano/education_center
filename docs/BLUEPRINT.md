# Education Center — Application Blueprint

> Status: **Draft v0.1** — this is the starting blueprint for the project. Sections marked
> **Assumption** were chosen as sensible defaults and should be confirmed or changed
> (see [§14 Open Questions](#14-open-questions)).

---

## 1. Vision

A single web platform for running an education center (tutoring center, language school,
training institute, or after-school academy). It replaces spreadsheets, paper registers and
scattered messaging with one system that handles **who is learning what, with whom, when,
and whether it has been paid for**.

### Goals

1. Admins can set up courses, classes, rooms and schedules in minutes.
2. Students (and parents) can find courses, enroll, pay, and follow their progress online.
3. Teachers can take attendance, post materials and record grades from any device.
4. Management gets real-time visibility into enrollment, attendance and revenue.

### Non-goals (for v1)

- Full video-conferencing (we integrate with Zoom/Google Meet links instead).
- Payroll / HR for staff.
- Native mobile apps (the web app is responsive and installable as a PWA).

---

## 2. Users & Roles

| Role | Who they are | Key capabilities |
|------|--------------|------------------|
| **Super Admin** | Owner / IT | Manage branches, admins, system settings, billing config |
| **Admin / Staff** | Front desk, coordinators | Manage courses, classes, schedules, enrollments, payments, reports |
| **Teacher** | Instructors, tutors | View own schedule, take attendance, upload materials, grade, message students |
| **Student** | Learners | Browse catalog, enroll, pay, view schedule, materials, grades, certificates |
| **Parent / Guardian** | Parents of minors | View linked children's schedule, attendance, grades, invoices; pay; message teachers |
| **Guest** | Public visitor | View course catalog, request info / trial class, register |

Permissions are enforced with **role-based access control (RBAC)** plus ownership checks
(e.g. a teacher can grade only the classes they teach; a parent can only see linked children).

---

## 3. Feature Modules

### 3.1 Identity & Accounts
- Email/password sign-up and login, optional Google sign-in.
- Email verification, password reset, optional 2FA for staff.
- Profiles: contact info, photo, date of birth, emergency contact.
- Parent ↔ student linking (one parent → many children; one child → up to 2 guardians).

### 3.2 Course Catalog
- **Course** = what is taught (e.g. "IELTS Preparation", "Grade 8 Math").
  - Title, description, subject/category, level, prerequisites, syllabus, cover image.
  - Default price, duration (weeks/sessions), max capacity.
- Public catalog with search and filters (subject, level, schedule, price, delivery mode).

### 3.3 Classes & Scheduling
- **Class** (a.k.a. section/batch) = one running instance of a course with a teacher,
  a room (or online link), a start/end date and a recurring timetable.
- Recurring session generation (e.g. "Mon & Wed 16:00–17:30 for 12 weeks").
- Conflict detection: teacher double-booking, room double-booking, student clash.
- Calendar views (day/week/month) per teacher, room, student and branch.
- Holidays / closure calendar that auto-skips sessions; reschedule & make-up sessions.

### 3.4 Enrollment
- Online enrollment by student/parent or manual enrollment by staff.
- Capacity management with **waitlist** and automatic promotion when a seat opens.
- Trial class booking for leads.
- Enrollment states: `pending_payment → active → completed | withdrawn | transferred`.
- Class transfer and withdrawal with configurable refund rules.

### 3.5 Attendance
- Teacher marks attendance per session: present / late / absent / excused.
- Optional QR-code self check-in.
- Automatic notification to parent when a student is absent.
- Attendance rate per student/class; alerts below a threshold (e.g. < 80 %).

### 3.6 Learning Content & Assessment
- Materials per class/session: files, links, videos, notes.
- Assignments with due dates, online submission and teacher feedback.
- Assessments/exams with scores; configurable grading scale and weighting.
- Progress report per student per class (attendance + grades + teacher comments).
- Completion certificates (PDF, generated automatically with verification code).

### 3.7 Billing & Payments
- Pricing: per course, per session package, monthly subscription; discounts and
  sibling/early-bird coupons.
- Invoices generated on enrollment or on a recurring schedule.
- Online payment via payment gateway (**Assumption:** Stripe; swap for a local gateway
  if needed), plus manual recording of cash/bank transfer.
- Receipts, partial payments, refunds, outstanding-balance reminders.

### 3.8 Communication
- In-app announcements (center-wide, per class).
- Direct messaging between teacher ↔ student/parent (staff can audit).
- Notifications via email and in-app; SMS/WhatsApp as a pluggable channel.
- Automated reminders: upcoming class, unpaid invoice, assignment due.

### 3.9 Lead / Inquiry Management (CRM-lite)
- Public inquiry & trial-class forms create **Leads**.
- Pipeline: `new → contacted → trial_booked → converted | lost`.
- Convert a lead to a student account in one click.

### 3.10 Reports & Dashboard
- Admin dashboard: active students, enrollments this month, attendance rate,
  revenue collected vs outstanding, class fill rates.
- Teacher dashboard: today's classes, pending grading, at-risk students.
- Exportable reports (CSV/PDF): enrollment, attendance, financial, teacher hours.

### 3.11 Administration
- Multi-branch support (each branch has rooms, staff, classes).
- Settings: academic terms, grading scales, notification templates, tax, currency,
  time zone, branding (logo/colors).
- Audit log of sensitive actions (grade changes, refunds, role changes).

---

## 4. Key User Flows

**Enroll in a course (student/parent)**
```
Browse catalog → Pick course → Choose class (schedule) → Log in / Register
  → (Parent selects child) → Review invoice → Pay → Enrollment active
  → Confirmation email + class appears in calendar
```

**Run a class session (teacher)**
```
Open "Today" → Select session → Mark attendance → Upload material / assignment
  → Absent students' parents notified automatically
```

**Monthly billing (admin)**
```
System generates recurring invoices → Emails sent → Payments reconcile via webhook
  → Unpaid after N days → Reminder → Dashboard shows outstanding balances
```

---

## 5. System Architecture

```
            ┌──────────────────────────────────────────────┐
            │                  Clients                      │
            │  Web app (responsive, PWA) — Students/Parents │
            │  Staff/Admin portal — same app, role-gated    │
            └──────────────────────┬───────────────────────┘
                                   │ HTTPS
            ┌──────────────────────▼───────────────────────┐
            │          Frontend: Next.js (React, TS)        │
            │   SSR catalog pages (SEO) + app dashboard     │
            └──────────────────────┬───────────────────────┘
                                   │ REST / JSON (OpenAPI)
            ┌──────────────────────▼───────────────────────┐
            │        Backend API: NestJS (Node, TS)         │
            │  Modules: auth, users, courses, classes,      │
            │  enrollment, attendance, assessment, billing, │
            │  messaging, leads, reports, admin             │
            └───┬──────────────┬──────────────┬────────────┘
                │              │              │
       ┌────────▼───┐  ┌───────▼──────┐  ┌────▼─────────────┐
       │ PostgreSQL │  │ Redis        │  │ Object storage   │
       │ (Prisma)   │  │ cache+queues │  │ (S3-compatible)  │
       └────────────┘  └───────┬──────┘  └──────────────────┘
                               │ BullMQ jobs
                       ┌───────▼──────────────────────────┐
                       │ Worker: emails, SMS, reminders,   │
                       │ invoice generation, PDF certs,    │
                       │ report exports                    │
                       └───────┬──────────────────────────┘
                               │
            External: Stripe · Email (SES/SendGrid) · SMS/WhatsApp · Google OAuth
```

**Assumption — architecture style:** a **modular monolith** (one deployable API with
clear module boundaries) plus one background worker. This is the simplest thing that
scales to thousands of students; modules can be split into services later if needed.

---

## 6. Technology Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Language | TypeScript (end to end) | One language, shared types between front and back |
| Frontend | Next.js + React, Tailwind CSS, shadcn/ui | SEO for catalog, fast UI development |
| Forms / validation | React Hook Form + Zod | Shared schemas client & server |
| Data fetching | TanStack Query | Caching, optimistic updates |
| Calendar UI | FullCalendar | Mature recurring-event views |
| Backend | NestJS | Modular structure, DI, guards for RBAC |
| ORM | Prisma | Type-safe DB access, migrations |
| Database | PostgreSQL 16 | Relational data, strong constraints |
| Cache / queues | Redis + BullMQ | Background jobs, rate limiting |
| Auth | JWT access + rotating refresh tokens (httpOnly cookies), Argon2 hashing | Secure, stateless |
| File storage | S3-compatible (AWS S3 / Cloudflare R2 / MinIO locally) | Materials & submissions |
| Payments | Stripe (Checkout + webhooks) | PCI handled by provider |
| Email | AWS SES or SendGrid, templates via React Email | Transactional mail |
| PDF | Playwright / pdf-lib | Certificates, invoices, reports |
| Testing | Vitest/Jest, Supertest, Playwright E2E | Unit → API → browser |
| Tooling | pnpm workspaces + Turborepo, ESLint, Prettier | Monorepo DX |
| Infra | Docker, GitHub Actions CI/CD | Reproducible builds |
| Hosting | Frontend on Vercel; API/worker on Render/Fly/AWS ECS; managed Postgres | Low ops overhead |
| Observability | Sentry, structured logs (pino), OpenTelemetry | Error & performance tracking |

---

## 7. Data Model

### 7.1 Entity overview

```
Branch 1─* Room
Branch 1─* Class
User 1─1 StudentProfile | TeacherProfile | GuardianProfile   (by role)
Guardian *─* Student                    (via GuardianLink)
Course 1─* Class
Teacher 1─* Class (primary)  / *─* Class (assistants via ClassTeacher)
Class 1─* ClassSchedule (recurrence rules)
Class 1─* Session
Student *─* Class                       (via Enrollment)
Session 1─* AttendanceRecord *─1 Enrollment
Class 1─* Material
Class 1─* Assessment 1─* Grade *─1 Enrollment
Class 1─* Assignment 1─* Submission *─1 Enrollment
Enrollment 1─* InvoiceItem *─1 Invoice 1─* Payment
Invoice *─1 User (payer)
Lead ─(converts to)→ User
Conversation 1─* Message ; Announcement ; Notification
AuditLog
```

### 7.2 Core tables (key fields)

| Table | Key fields |
|-------|------------|
| `users` | id, email (unique), password_hash, first_name, last_name, phone, role, status, locale, created_at |
| `student_profiles` | user_id, date_of_birth, school, grade_level, notes |
| `teacher_profiles` | user_id, bio, subjects[], hourly_rate |
| `guardian_links` | guardian_id, student_id, relationship, is_primary |
| `branches` | id, name, address, timezone, phone |
| `rooms` | id, branch_id, name, capacity |
| `courses` | id, code, title, description, subject, level, default_price, total_sessions, capacity, is_published |
| `classes` | id, course_id, branch_id, teacher_id, room_id, name, start_date, end_date, capacity, delivery_mode (in_person/online/hybrid), meeting_url, status |
| `class_schedules` | id, class_id, weekday, start_time, end_time |
| `sessions` | id, class_id, starts_at, ends_at, room_id, status (scheduled/held/cancelled/rescheduled), topic |
| `enrollments` | id, class_id, student_id, status, enrolled_at, withdrawn_at, price, discount_id — unique(class_id, student_id) |
| `waitlist_entries` | id, class_id, student_id, position, created_at |
| `attendance_records` | id, session_id, enrollment_id, status, marked_by, marked_at, note — unique(session_id, enrollment_id) |
| `materials` | id, class_id, session_id?, title, type, file_key/url, published_at |
| `assignments` | id, class_id, title, instructions, due_at, max_score |
| `submissions` | id, assignment_id, enrollment_id, file_key, text, submitted_at, score, feedback |
| `assessments` | id, class_id, title, type, weight, max_score, date |
| `grades` | id, assessment_id, enrollment_id, score, comment, graded_by |
| `certificates` | id, enrollment_id, code (unique), issued_at, file_key |
| `invoices` | id, number, payer_id, branch_id, status (draft/open/paid/void/overdue), due_date, subtotal, discount, tax, total, currency |
| `invoice_items` | id, invoice_id, enrollment_id?, description, quantity, unit_price |
| `payments` | id, invoice_id, amount, method (card/cash/transfer), provider_ref, status, paid_at |
| `discounts` | id, code, type (percent/fixed), value, valid_from, valid_to, max_uses |
| `leads` | id, name, email, phone, interest_course_id, source, status, assigned_to, notes |
| `announcements` | id, scope (global/branch/class), scope_id, title, body, published_at |
| `conversations` / `messages` | participants, sender_id, body, read_at |
| `notifications` | id, user_id, type, payload (jsonb), channel, sent_at, read_at |
| `audit_logs` | id, actor_id, action, entity, entity_id, before (jsonb), after (jsonb), at |

All tables get `id` (UUID), `created_at`, `updated_at`; soft delete (`deleted_at`) on
user-facing records. Money stored as integer minor units (cents) + currency code.
Timestamps stored in UTC; displayed in the branch time zone.

---

## 8. API Design

- REST, JSON, versioned under `/api/v1`, documented with **OpenAPI** (auto-generated by NestJS).
- Cursor-based pagination, `?filter` and `?sort` conventions, consistent error envelope:
  `{ "error": { "code": "CLASS_FULL", "message": "...", "details": {} } }`.

| Area | Representative endpoints |
|------|--------------------------|
| Auth | `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `POST /auth/password/forgot` |
| Users | `GET /me`, `PATCH /me`, `GET /users`, `POST /users` (staff), `POST /guardians/:id/students` |
| Courses | `GET /courses`, `GET /courses/:id`, `POST /courses`, `PATCH /courses/:id` |
| Classes | `GET /classes`, `POST /classes`, `POST /classes/:id/sessions/generate`, `GET /classes/:id/roster` |
| Sessions | `GET /sessions?from&to&teacherId`, `PATCH /sessions/:id` (reschedule/cancel) |
| Enrollment | `POST /enrollments`, `POST /enrollments/:id/withdraw`, `POST /enrollments/:id/transfer`, `POST /classes/:id/waitlist` |
| Attendance | `PUT /sessions/:id/attendance` (bulk), `GET /students/:id/attendance` |
| Learning | `POST /classes/:id/materials`, `POST /assignments/:id/submissions`, `PUT /assessments/:id/grades` |
| Billing | `GET /invoices`, `POST /invoices/:id/checkout`, `POST /payments` (manual), `POST /webhooks/stripe` |
| Messaging | `GET /conversations`, `POST /conversations/:id/messages`, `POST /announcements` |
| Leads | `POST /public/inquiries`, `GET /leads`, `POST /leads/:id/convert` |
| Reports | `GET /reports/dashboard`, `GET /reports/attendance.csv`, `GET /reports/revenue` |
| Files | `POST /uploads/presign` (direct-to-S3 upload) |

---

## 9. Frontend Structure (pages)

```
/                        Public landing page
/courses                 Catalog (search/filter)       — public, SSR
/courses/[slug]          Course detail + open classes  — public, SSR
/login  /register  /forgot-password

/app                     Role-aware dashboard
/app/schedule            Calendar
/app/classes             My classes (teacher/student)
/app/classes/[id]        Class hub: sessions, roster, materials, assignments, grades
/app/sessions/[id]/attendance
/app/children            Parent: linked children overview
/app/billing             Invoices & payments
/app/messages            Inbox
/app/profile

/admin                   Admin dashboard (KPIs)
/admin/courses  /admin/classes  /admin/rooms  /admin/branches
/admin/students /admin/teachers /admin/guardians
/admin/enrollments  /admin/waitlists
/admin/invoices /admin/payments /admin/discounts
/admin/leads    /admin/reports  /admin/settings  /admin/audit-log
```

UX principles: mobile-first for students/parents/teachers (attendance on a phone),
data-dense tables for admins, WCAG 2.1 AA accessibility, i18n-ready (**Assumption:**
English first, additional languages later).

---

## 10. Repository Layout

```
education_center/
├── apps/
│   ├── web/                 # Next.js frontend
│   ├── api/                 # NestJS REST API
│   └── worker/              # BullMQ background jobs
├── packages/
│   ├── db/                  # Prisma schema, migrations, seed
│   ├── shared/              # Zod schemas, types, constants (roles, statuses)
│   ├── ui/                  # Shared React components
│   └── config/              # ESLint, TS, Tailwind presets
├── docs/
│   ├── BLUEPRINT.md         # this document
│   └── adr/                 # architecture decision records
├── infra/
│   ├── docker-compose.yml   # postgres, redis, minio, mailpit for local dev
│   └── deploy/
├── .github/workflows/       # CI: lint, typecheck, test, build; CD on main
├── package.json  pnpm-workspace.yaml  turbo.json
└── README.md
```

---

## 11. Non-Functional Requirements

**Security & privacy**
- OWASP Top 10 mitigations: input validation (Zod/class-validator), parameterized queries
  (Prisma), CSRF protection on cookie auth, strict CORS, rate limiting on auth endpoints.
- Passwords hashed with Argon2id; staff 2FA (TOTP).
- Row-level authorization checks in every service method (not just route guards).
- Minors' data: guardian consent flag, minimal data collection, data export & deletion on
  request (GDPR/local privacy law compliant).
- Files served via short-lived signed URLs; card data never touches our servers.
- Encrypted at rest (managed DB/storage) and in transit (TLS).

**Performance & scale (v1 targets)**
- Up to ~5,000 active students, ~300 concurrent users, 10 branches.
- p95 API latency < 300 ms; catalog pages cached at the edge.

**Reliability**
- Daily automated DB backups with point-in-time recovery; tested restore.
- Idempotent payment webhooks; background jobs with retries and dead-letter queue.
- 99.5 % availability target.

**Quality**
- Unit tests for domain logic (scheduling, billing, grading) — ≥ 80 % coverage there.
- API integration tests against a real Postgres (Testcontainers).
- Playwright E2E for the critical flows: register → enroll → pay; take attendance; grade.
- CI blocks merge on lint, typecheck and test failures.

---

## 12. Delivery Roadmap

| Phase | Duration (est.) | Scope | Exit criteria |
|-------|-----------------|-------|---------------|
| **0. Foundation** | 1–2 weeks | Monorepo, CI, Docker dev env, DB schema v1, auth & RBAC, admin shell | Users can register/login with roles; CI green |
| **1. Core academics (MVP)** | 3–4 weeks | Courses, classes, schedules, session generation, calendar, manual enrollment, attendance | Admin runs a real term without spreadsheets |
| **2. Self-service & billing** | 3 weeks | Public catalog, online enrollment, waitlist, invoices, Stripe payments, parent portal | Parent can enroll and pay a child end-to-end |
| **3. Learning & communication** | 3 weeks | Materials, assignments, grades, progress reports, announcements, messaging, email notifications | Teachers run classes fully in-app |
| **4. Growth & insight** | 2–3 weeks | Leads/CRM, trial classes, dashboards, exports, certificates, multi-branch | Management dashboard in daily use |
| **5. Hardening & launch** | 2 weeks | Security review, load test, backups, monitoring, docs, data migration from old tools | Production launch |

**MVP = Phases 0–2** (~8 weeks for a small team of 2–3 developers).

---

## 13. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Scheduling edge cases (time zones, DST, holidays) | Store UTC, use rrule-style recurrence, dedicated test suite |
| Payment reconciliation errors | Webhook idempotency keys, nightly reconciliation job, manual override with audit log |
| Scope creep (LMS features) | Strict phase gates; integrate external tools (Zoom, Google Drive) before building |
| Data migration from existing spreadsheets | CSV import tools for students, classes and balances in Phase 5 |
| Handling children's data | Privacy review, consent capture, least-privilege access |

---

## 14. Open Questions

Please confirm or correct these so the blueprint can move from draft to v1.0:

1. **Type of center** — tutoring (K-12), language school, professional training, or other?
   This affects grading scales, terminology ("class" vs "batch"), and the parent portal.
2. **Delivery mode** — in-person only, online, or hybrid?
3. **Single or multiple branches** at launch?
4. **Payment gateway & currency** — Stripe, or a local provider? Which country/tax rules?
5. **Languages** the UI must support (and any right-to-left requirement)?
6. **Messaging channels** — is WhatsApp/SMS required at launch?
7. **Existing data/tools** to migrate from (Excel, Google Sheets, another system)?
8. **Tech stack preferences** — any team constraints (e.g. Python/Django, Laravel, mobile app)?
9. **Hosting & budget** constraints?
10. **Team size & target launch date?**
