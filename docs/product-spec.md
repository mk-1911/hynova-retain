# Product Specification

HYNOVA Retain — member retention for gyms.

This document describes **what the product does and why**. For how it is built, see
[architecture.md](architecture.md). For the rules the database enforces, see
[database.md](database.md).

---

## 1. The problem

A gym already owns the data that predicts churn. It does not act on it.

- **Membership software** knows a member's billing status. It does not know they stopped coming.
- **Attendance data** shows a drop. It does not say who is at risk, or what to do about it.
- **Front-desk staff** notice a quiet member — inconsistently, and usually late.
- **Email and SMS retention campaigns** underperform badly, because gym members do not
  read email.

By the time churn appears in revenue, the member is already gone. Retention has to happen
*before* the cancellation, which means somebody has to notice and act.

### Where this fits

HYNOVA Retain does not replace the CRM or the membership system. It is the **retention
decision layer** on top of them: it reads attendance and membership state, decides who is
at risk, initiates contact on the channel members actually answer, and holds a named human
responsible for the outcome.

---

## 2. Users and roles

Three roles, enforced in the database as well as the application.

| Role | Sees | Can do |
|---|---|---|
| **Owner** | Everything in the gym | Full configuration, trainer management, billing settings |
| **Manager** | Everything in the gym | Day-to-day operations; same case and member access as owner |
| **Trainer** | **Only their own members and cases** | Act on assigned cases; no gym configuration, no member roster |

The trainer view is not a filtered version of the staff view. It is a separate
information architecture with its own shell, sidebar and navigation, because the job is
different: staff administer, trainers coach and close cases.

**Scope rule.** A trainer reaches a resource if and only if the resource's member
(a) belongs to the trainer's gym and (b) is assigned to that trainer. Both conditions are
enforced in SQL, not in the UI. A trainer requesting a case outside their scope gets
`404 Not Found` rather than `403` — the API will not confirm that an out-of-scope case
exists.

---

## 3. Core concepts

### Risk score

A 0–100 score per member, recomputed on every Detection cycle from four signals:

| Signal | What it measures |
|---|---|
| **Attendance** | Recency and frequency against the member's own normal schedule |
| **Trend** | Direction of travel — is attendance falling, stable, or recovering |
| **Membership** | Time remaining, plan state, lapse risk |
| **History** | Previous retention cases and whether they were resolved |

The four sub-scores are stored on `risk_scores` for explainability. The composite is what
drives the case.

### Risk levels

Bands, defined per gym in `retention_settings`:

| Level | Default threshold | Meaning |
|---|---|---|
| `healthy` | 0–29 | Nothing to do |
| `watch` | 30–49 | Worth watching |
| `at_risk` | 50–69 | Active retention needed |
| `high_risk` | 70–84 | Urgent |
| `critical` | 85–100 | At the point of leaving |

The database enforces that the four thresholds are **strictly increasing**, so a
misconfiguration is impossible rather than merely unlikely.

### Retention case

The unit of work. Created when a member crosses a configured threshold. Carries the risk
score and level, a plain-language `reason`, the owning trainer, and a status.

**A case always states why it exists.** The `reason` is written in member-facing terms
("No visit in 18 days"), not as a score dump. A trainer can judge whether the reasoning is
sound, and that is what makes them willing to act on it.

### Ownership follows the member

`members.assigned_trainer_id` is the **single source of truth** for who owns a case.
Reassigning a member moves their active cases with them, so "My Members" and "My Cases"
never disagree.

Resolved and closed cases keep the trainer who actually handled them — history is not
rewritten.

This is enforced by two database triggers. See [database.md](database.md#case-ownership).

---

## 4. The retention loop

The product's whole value is in one closed loop. Each stage exists because the previous one
alone is insufficient.

### 4.1 Detect

**Retention Detection** runs every 5 minutes. For every active member it loads attendance,
membership and retention history, computes the four sub-scores, combines them, maps the
result to a risk level, and compares against that gym's thresholds.

A member who crosses a threshold and does not already have an open case gets one. The
`Case Already Exists?` check runs on every cycle, so a member cannot accumulate duplicate
cases for the same episode.

### 4.2 Decide

Whether to actually contact someone is a separate decision from whether they are at risk.
Outreach requires **all** of:

- The case is `open` (not resolved, closed, or already worked).
- The member has `contact_allowed = true`.
- The member has an active membership.
- The cooldown since the last outreach on this case has expired.
- The per-case automated outreach cap has not been reached.
- No trainer has already taken the case in hand.

`contact_allowed` is a per-member consent switch. When it is off, **every** entry query
excludes the member structurally — it is not a flag checked after the fact.

### 4.3 Reach out

An AI writes a short WhatsApp message in the member's own language. It is sent through the
Evolution API gateway and recorded in `messages` with the provider's message id.

The message is deliberately short, names the member, asks exactly one question, and never
mentions risk, scoring, automation, or inactivity. The guardrails are in
[ai.md](ai.md#guardrails).

### 4.4 Listen

If the member replies, the Inbound Response Engine receives it, saves the interaction, and
asks the AI to classify intent, sentiment, confidence, and — critically —
**whether a human needs to intervene**.

### 4.5 Escalate

Anything involving injury, a health concern, a serious complaint, strong dissatisfaction,
stated cancellation intent, or anything unclear is routed to the member's trainer with the
full case context attached, not a bare notification.

### 4.6 Close the loop

The trainer acts. Every action is an immutable audit entry with actor and timestamp. The
case cannot sit open silently — the Trainer Workflow re-reminds until it is actioned.

---

## 5. Trainer case actions

Four actions, each with defined semantics.

| Action | Effect on the case | Audit entry | Available when |
|---|---|---|---|
| **Contact** | `status → in_progress` | `trainer_contacted` | Case open or in progress |
| **Resolve** | `status → resolved`, sets `resolved_at` | `case_resolved` | Case open or in progress |
| **Snooze** | `status → in_progress`, sets `snoozed_until = now + N days` | `case_snoozed` | Case open or in progress |
| **Reassign** | Moves the **member** to a new trainer; reopens the case and clears any snooze | `trainer_reassigned` | Case open or in progress |

### Why snooze keeps the case in progress

A snooze pauses *outreach*, it does not park the case. The date lives in `snoozed_until`,
and every workflow that might contact a member honours it. This keeps the case visible in
the trainer's work queue so it cannot be forgotten, which is the entire point of the
accountability loop.

A bare `status = 'snoozed'` without a date does the opposite: it removes the case from
every queue, including the one that would remind anyone about it. The UI therefore does not
offer that as a snooze — it is only reachable as a raw status edit.

### Why reassign moves the member

Changing `retention_cases.assigned_trainer_id` directly would create a case owned by
someone other than the member's trainer, and the two would immediately disagree. Reassigning
the member moves their whole book — future cases included — and a trigger propagates the
change to active cases.

The reassignment target is validated twice: the API checks it is an active trainer in the
same gym, and the workflow independently re-checks before writing.

---

## 6. Configuration

Per gym, editable from **Settings → Automation** and stored in `retention_settings`.

| Setting | Live value | What it controls |
|---|---|---|
| `inactivity_threshold_days` | 14 | Days without a visit before a member counts as drifting |
| `watch_threshold` | 30 | Risk score at which a member becomes "watch" |
| `at_risk_threshold` | 50 | …and "at risk" |
| `high_risk_threshold` | 70 | …and "high risk" |
| `critical_threshold` | 85 | …and "critical" |
| `followup_delay_days` | 4 | How long a case waits before follow-up |
| `max_automated_outreach` | 3 | Hard cap on automated messages per case |
| `cooldown_hours` | 48 | Minimum hours between automated messages on a case |

**Cooldown and the outreach cap are the two settings that directly govern brand risk.** A
gym that fears being spammed tightens them; a gym chasing volume loosens them. The defaults
are conservative: 3 messages, 48 hours apart.

---

## 7. Feature inventory

### Gym staff

| Area | Capability |
|---|---|
| **Overview** | Gym-wide health: members at risk, retention performance, recent activity |
| **Today** | A work queue of what needs attention now |
| **Members** | Full roster, profiles, assignment, status, attendance history |
| **Memberships** | Plans, start/end dates, renewal, value at risk |
| **Retention cases** | Every case, filterable by status and risk, with full history |
| **Case detail** | Risk, reason, attendance trend, message history, audit trail, all four actions |
| **Trainers** | Roster, assignment, invites, account activation, status |
| **Analytics** | Risk distribution, attendance trend, retention trend, case status, message performance, trainer workload, membership expiries |
| **Automation** | Every `retention_settings` value |
| **Settings** | Gym profile, language, appearance |

### Trainers

| Area | Capability |
|---|---|
| **My Work** | Cases requiring action, scoped to the trainer's own members |
| **My Members** | Roster and profiles of assigned members only |
| **My Cases** | Assigned cases only |
| **Case actions** | Contact, resolve, snooze (with duration), reassign (with trainer picker) |
| **Notifications** | Escalations and reminders |
| **Account** | Profile, password, avatar |

---

## 8. Non-functional requirements

These are properties the product is expected to hold, and how they are currently met.

| Requirement | Current state |
|---|---|
| No unauthenticated automation entry point | **Not met** — both webhooks have `authentication: none`. See deployment.md |
| Automated outreach never exceeds the configured cap | Met — enforced in SQL per case |
| Outreach never repeats inside the cooldown | Met — enforced in SQL per case |
| No contact to a member with `contact_allowed = false` | Met — excluded structurally in every entry query |
| Audit entries cannot be forged | Met — actor derived from the verified token; `performed_by_id` FK-constrained |
| Case ownership cannot drift from member ownership | Met — two database triggers |
| Invalid states are rejected | Met — 93 CHECK constraints |
| Automation survives transient database failure | **Not met** — 0 of 25 Postgres nodes set `retryOnFail` |
| Actions are idempotent under double-click | Partial — buttons disable while pending; state-conflicting actions return `409` |
| Regression suite protects changes | **Not met** — no automated tests exist |

---

## 9. Deliberate non-goals

Stated so they are not mistaken for gaps.

- **Not a membership billing system.** It reads membership state; it does not charge.
- **Not a CRM.** No sales pipeline, no lead scoring, no campaigns.
- **Not a replacement for the front desk.** The product's premise is that automation finds
  the signal and humans provide the judgement — not that software replaces the conversation.
- **Not multi-tenant yet.** The schema is gym-scoped throughout and the deployment is
  single-gym, but multi-site hardening has not been designed or tested.
- **Not a messaging platform.** It owns WhatsApp outbound through a gateway; it does not
  run a helpdesk inbox.

---

## 10. Language support

The member-facing UI and the AI prompt layer are **not aligned**, and it matters commercially.

- **The application offers two languages**: English and Arabic
  (`frontend/src/lib/languages.ts`).
- **The outreach AI prompt instructs 13 language codes** (en, ar, es, fr, de, it, pt, tr,
  ru, zh, ja, ko, plus a fallback).

A member can therefore only be *recorded* as English or Arabic through the UI, but a code
written to the database by another route would be honoured by the AI. Treat **English and
Arabic as the supported set** until `LANGUAGES` is extended — and extend it if the
commercial plan needs more.