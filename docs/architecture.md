# Architecture

How HYNOVA Retain is put together, and why it is put together that way.

For product behaviour see [product-spec.md](product-spec.md). For the schema see
[database.md](database.md). For the automation layer see [workflows.md](workflows.md).

---

## 1. System shape

```
                    ┌───────────────────────────────────────────────┐
                    │              PostgreSQL (Supabase)           │
                    │   16 tables · 3 triggers · 93 CHECK constraints│
                    │   retention_cases, members, trainers, users,   │
                    │   messages, interactions, retention_actions,  │
                    │   attendance, memberships, risk_scores, …      │
                    └───────┬───────────────────────────┬───────────┘
                            │ SQL (pg, pooled)          │ SQL (n8n node)
                            │                           │
        ┌───────────────────▼──────────┐      ┌─────────▼──────────────────┐
        │   Fastify API  (port 3000)    │      │      n8n  (port 5678)     │
        │                              │─────▶│  4 × scheduled (5 min)    │
        │  · JWT auth                   │ HTTP │  2 × webhook              │
        │  · SQL-level scoping          │      │  · AI agents (Groq)        │
        │  · n8n proxy                  │◀─────│                          │
        │  · REST, 69 endpoints         │      └─────────┬──────────────────┘
        └───────────────┬──────────────┘                │
                        │ REST + JWT                     │ HTTP
                        │                      ┌─────────▼──────────────────┐
        ┌───────────────▼──────────────┐      │  Evolution API (port 8080) │
        │   React SPA  (Vite, :5173)   │      │  instance: hynova-retain   │
        │                              │      └─────────┬──────────────────┘
        │  staff shell  /app/*         │                │
        │  trainer shell /trainer/*    │                ▼
        └──────────────────────────────┘           WhatsApp Business
```

### The single most important structural decision

**The browser never talks to n8n.**

The trainer UI calls `POST /retention/cases/:caseId/case-action` on the Fastify API. The
API validates, authorises, derives the acting identity, and only then calls the n8n
production webhook URL — which lives in `backend/.env` and is never sent to the client.

This buys four things at once:

1. **The webhook URL stays secret.** Anyone who can reach port 5678 can currently post to
   the webhooks directly (see [known issues](deployment.md#known-issues)); keeping the URL
   server-side means there is exactly one caller to reason about, and adding header
   authentication later is a backend-only change.
2. **Identity is trustworthy.** The workflow authorises an action by matching a
   `trainer_id` against the case's assigned trainer. A browser-supplied trainer id would be
   forgeable. The API derives it from the verified JWT instead, so the value the workflow
   checks is one the client never controlled.
3. **Errors are mappable.** n8n speaks in webhook status codes; the UI wants Fastify
   responses with messages a trainer can act on. The proxy translates between them.
4. **Timeouts are bounded.** The webhook is a local HTTP call, but a hung n8n would hang the
   UI. The proxy aborts at 15 s and returns `504`.

---

## 2. Request lifecycle

### A trainer loads their case list

```
Browser
  │  GET /retention/cases?status=open          Authorization: Bearer <jwt>
  ▼
Fastify
  ├─ @fastify/jwt verifies the signature
  ├─ auth.ts checks user_id / gym_id / role are well-formed,
  │  and re-reads users.active from the database  ← deactivation is immediate
  ▼
retention.ts
  ├─ requireAuth      → rejects anything without a valid session
  ├─ getTrainerScope  → returns the trainer's id, or null for staff
  ▼
SQL
     SELECT … FROM retention_cases rc
       JOIN members m ON m.id = rc.member_id
      WHERE m.gym_id = $1                        -- the user's gym
        AND ($2::uuid IS NULL                    -- null for staff: no trainer filter
             OR m.assigned_trainer_id = $2)      -- trainers: own members only
```

Both scoping conditions live in the query. A trainer cannot widen their scope by changing
request parameters, because the values come from the token, not the request.

### A trainer performs a case action

```
Browser  POST /retention/cases/:id/case-action  { action: "snooze", snooze_days: 7 }
  │
  ▼  the body carries NO trainer id
Fastify
  ├─ requireAuth + requireCaseScope            → 404 if the case is not theirs
  ├─ validate action ∈ {contact,resolve,snooze,reassign}
  ├─ validate snooze_days is an integer in [1, 90]
  ├─ snooze/reassign only → 409 if the case is already resolved/closed
  ├─ reassign → verify target trainer is active and in the same gym
  ├─ trainer_id = request.user.trainer_id      ← server-derived
  ├─ reject if N8N_CASE_ACTION_WEBHOOK_URL is unset or is a /webhook-test/ URL
  ▼  POST http://localhost:5678/webhook/gym-retention-case-action   (15 s timeout)
n8n  Case Resolution workflow
  ├─ Validate Case Action        → shape, allowed actions, per-action required fields
  ├─ Load Case + Trainer         → the case, its member, and the trainer's active flag
  ├─ Trainer Authorized?        → assigned_trainer_id === trainer_id AND trainer active
  ├─ Determine Action            → switch to one of four branches
  ├─ <action mutation> + <audit insert>
  ▼  { success: true, message, case_id, action }
Fastify
  └─ 200 { success: true, action, case_id }     ← only if success === true
```

**Success is reported only after both the mutation and the audit insert have run.** If the
mutation matches no rows the workflow produces no items, so the audit node never executes
and no response is produced — the proxy surfaces that as a failure rather than a silent
success.

---

## 3. Layers

### `frontend/` — React 19 + TypeScript + Vite

```
src/
├── App.tsx              route table; every page is lazy()
├── components/
│   ├── layout/          AppShell, TrainerShell, sidebars, topbars
│   ├── ui/                15 primitives — Button classes, Surface, DataTable, Modal…
│   ├── ProtectedRoute   role gate for /app/*   (owner, manager)
│   └── TrainerRoute     role gate for /trainer/*
├── context/             auth (JWT + user), theme (light/dark)
├── hooks/               10 data-fetching hooks
├── lib/                 api client, formatters, design tokens, CSV export
├── pages/               23 route components
├── services/            typed endpoint wrappers
└── index.css            Tailwind 4 + component layer
```

**Two shells, one design system.** `/app/*` renders `AppShell` for owners and managers;
`/trainer/*` renders `TrainerShell` for trainers. Separate navigation because the jobs
differ — but the same `ui/` primitives, so the product reads as one system.

**Data fetching** is TanStack Query throughout. Mutations invalidate by query key
(`retention-case`, `retention-case-actions`, `retention-cases`) so every view of a case
refreshes after an action.

**Route-level code splitting** keeps the initial payload at ~293 kB (~89 kB gzipped); pages
download on first visit.

**The Tailwind layer order matters.** `index.css` puts component classes in
`@layer components` and the `a { color: inherit }` reset inside `@layer base`. An unlayered
rule outranks every layered one regardless of specificity, so an unlayered anchor reset
silently beat `.btn-primary`'s `text-white` and left every `<Link>`-styled button with dark
text on a teal background. Keep resets layered.

### `backend/` — Fastify 5 + TypeScript

```
src/
├── server.ts            entrypoint; PORT from env
├── app.ts               Fastify instance, CORS, helmet, JWT, rate limit, route registration
├── db/
│   ├── postgres.ts      pooled connection
│   ├── ensure-schema.ts idempotent boot-time schema guard
│   └── migrations/      3 idempotent .sql files
├── hooks/
│   ├── auth.ts          verify(), requireAuth, role helpers
│   └── scope.ts         requireMemberScope, requireCaseScope
├── routes/              14 files, 69 endpoints
└── services/            email transport (Brevo / Resend)
```

**Plugins**, in order: `@fastify/cors` (origin from `FRONTEND_URL`), `@fastify/helmet`,
`@fastify/jwt`, `@fastify/rate-limit`, then routes.

**`ensureSchema()` runs at boot** and is idempotent. It adds columns, creates constraints
and triggers with `DROP … IF EXISTS` before `CREATE`, and reconciles any pre-existing
ownership drift. A fresh deployment needs no separate migration step; the `.sql` files in
`db/migrations/` exist so the same changes can be applied by hand or by a migration runner.

**Conventions** observed throughout the route layer: `process.env.X?.trim()`, explicit
response objects (`{ error }` / `{ success }`), `isValidUuid` before any id reaches SQL, and
client-supplied identity fields rejected rather than trusted.

### `n8n/` — automation

Six active workflows. Four scheduled on a 5-minute cycle, two webhooks. Each scheduled
workflow is a **checker**: it re-evaluates eligibility every cycle and acts at most once
per case.

The Postgres node uses `queryReplacement` with `$1, $2` binds rather than string
interpolation, so member names and case reasons cannot alter query shape.

Full detail in [workflows.md](workflows.md).

---

## 4. Data flow

### Detection → case

```
Schedule Trigger (5 min)
  → Get Active Gyms
  → Get Active Members            (contact_allowed, active membership)
  → Get Active Memberships        (end_date, status)
  → Get Attendance Metrics        (last 30 days)
  → Calculate Attendance Score    ┐
  → Calculate Attendance Trend    │
  → Get Membership Risk           ├ Code nodes, runOnceForEachItem
  → Calculate Membership Risk     │
  → Get Retention History         │
  → Calculate History Risk        │
  → Calculate Final Risk Score    ┘
  → Get Retention Settings        (per-gym thresholds)
  → Determine Risk Level
  → Retention Eligible?
  → Check Existing Retention Case?
  → Create Retention Case         INSERT retention_cases
  → Record Case Creation          INSERT retention_actions
```

### Outreach → message

```
Schedule Trigger (5 min)
  → Get Eligible Retention Cases   status='open' AND contact_allowed
  → Load Member + Trainer Context
  → Get Retention Settings
  → Check Outreach Limits           per-case cap
  → Check Outreach Cooldown         per-case last-sent
  → Normalize Cooldown              runOnceForEachItem
  → Cooldown Passed?
  → Outreach Allowed?
  → Check Outreach Eligibility      final gate
  → Determine Outreach Action
  → Need Trainer Escalation?
  → Generate Retention Message     AI (Groq)
  → Create Message Record          INSERT messages (pending)
  → Send WhatsApp Message          POST evolution_api
  → WhatsApp Send Successful?
  → Update Message Status          → sent / failed
  → Log Retention Action           INSERT retention_actions
  → Send Trainer Alert             when escalating
```

### Inbound reply → reply or escalation

```
Webhook  POST /webhook/hynova-retain/inbound
  → Identify Member            (by phone)
  → Find Active Retention Case
  → Reserve Interaction        deduplication guard
  → Save Interaction           INSERT interactions
  → AI Analyze Reply           intent / sentiment / requires_human
  → Parse AI Analysis
  → Determine Response Action
      ├── requires_human → Get Trainer Details → Send Trainer Alert → Log Escalation
      └── else           → Generate Member Response (AI)
                          → Create Message Record
                          → Send WhatsApp Message
                          → Update Message Status
                          → Log Retention Action
```

### Trainer case action

```
Browser → Fastify → POST /webhook/gym-retention-case-action → n8n → database → response
```

Described in full in [request lifecycle](#a-trainer-performs-a-case-action).

---

## 5. Security model

### Authentication

| Concern | Mechanism |
|---|---|
| Password storage | bcrypt via `bcrypt` package; never returned by any endpoint |
| Session | JWT (`@fastify/jwt`), subject is `users.id` |
| Revocation | `users.active` re-read from the database on every request — deactivation is immediate, not deferred to token expiry |
| Trainer link | `users.trainer_id`, with a database CHECK that a `trainer` role must have one and no other role may |
| Cross-gym trainer | Composite FK `(trainer_id, gym_id) → trainers (id, gym_id)` — a trainer account cannot belong to another gym |

### Authorization

Two mechanisms, deliberately layered:

1. **Role check** — `requireAuth` plus `allowRoles` at the route or shell.
2. **Resource scope** — `requireCaseScope` / `requireMemberScope`, both SQL predicates that
   combine the user's gym with, for trainers, their own assignment.

Out-of-scope returns `404`, not `403`. This is deliberate: a `403` confirms the resource
exists, which leaks information about other trainers' members.

### Audit integrity

`retention_actions.performed_by_id` references `users(id)`. The API derives the value from
the verified token and **rejects** a client-supplied one. An audit row therefore cannot be
attributed to somebody who did not perform the action — the failure mode is a rejected
request, not a forged row.

`performed_by` is additionally constrained to a fixed label set, so the human-readable
actor cannot be invented either.

### Trust boundaries

| Boundary | Rule |
|---|---|
| Browser → API | JWT; trainer identity never accepted from the client |
| API → database | Parameterised queries; `isValidUuid` before any id |
| API → n8n | The only caller. URL server-side only |
| n8n → database | `queryReplacement` binds; no string interpolation |
| n8n → Evolution API | Internal Docker network (`gym-net`), not host-exposed to the browser |
| Client → Evolution API | **None.** Not exposed by the frontend at any point |

### Current gaps

Stated plainly, because they are the ones that matter before a customer connects:

- **Both webhooks are unauthenticated** (`authentication: none`). Anything that can reach
  port 5678 can inject a reply or a trainer action.
- **Database credentials and an n8n API token are committed** in plaintext files under
  `Supabase/` and `n8n/`. See [deployment.md](deployment.md#credential-hygiene).
- **No rate limiting on the webhooks**, which compounds the point above.

---

## 6. Key design decisions

### Case ownership lives on the member, not the case

`members.assigned_trainer_id` is authoritative; `retention_cases.assigned_trainer_id` is a
trigger-maintained cache.

The alternative — treating the case as independently assignable — produces a trainer whose
member list and case list disagree, and cases owned by someone who has never spoken to the
member. The cost of this decision is that n8n must write `members`, not `retention_cases`,
for reassignment, which is why the Case Resolution workflow's `Reassign Case` node updates
the member and lets the trigger propagate.

Worth knowing: the previous implementation wrote the case column directly and **did not
work at all** — the `BEFORE UPDATE` trigger on `retention_cases` overwrote the value from
the member on every write, so the reassignment was silently reverted. Ownership drift
stayed at zero only because reassignment was a no-op.

### Scoping in SQL, not in the UI

A trainer's visible set is a predicate in the query, not a filter applied to a full result
set. There is no code path that fetches "all cases then hides some", so there is no way to
accidentally leak.

### Rejecting client-supplied identity

The API does not "prefer" the token over the body — it does not accept an identity field in
the body at all. A trainer that must be told which trainer they are is a design smell; the
server should know.

### Automation as checkers, not senders

Each 5-minute cycle re-evaluates eligibility and acts at most once per case. The alternative
— a workflow that fires and sends — makes cooldown enforcement depend on the schedule being
exactly right, and a single misfire becomes spam. As a checker, being late is harmless and
being run twice changes nothing.

### Fail closed at the proxy

A `200` from the API means the workflow reported `success: true`. A `200` that omits
`success`, a non-JSON body, a timeout, or any non-2xx status all become `502`/`504`. The UI
never sees a success it cannot substantiate.