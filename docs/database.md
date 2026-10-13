# Database Reference

PostgreSQL (Supabase, remote) with `pgvector` enabled.

**16 application tables, 5 legacy backup tables, 5 triggers, 93 CHECK constraints, 49
indexes.** Extracted from the live database on 2026-10-13.

Status vocabulary in this document is enforced by CHECK constraints — an invalid value is
rejected by Postgres, not by application code.

---

## 1. Entity relationships

```
                          ┌──────────┐
                          │  gyms    │
                          └────┬─────┘
              ┌────────────────┼────────────────┬──────────────┐
              │                │                │              │
       ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼───────┐      │
       │   members   │  │  trainers   │  │retention_    │      │
       └──┬────┬─────┘  └──────┬──────┘  │  settings    │      │
          │    │               │         └──────────────┘      │
          │    │               │ 1                            │
          │    │               ▼                              │
          │    │         ┌──────────┐   1:1 (composite FK)     │
          │    │         │  users   │──────────────────────────┘
          │    │         └──────────┘
          │    │                    ▲
          │    │  assigned_trainer_id│performed_by_id
          │    └────────────────────┼──────────┐
          │                         │          │
   ┌──────▼──────┐   ┌──────┐  ┌────▼──────────▼───┐  ┌───────┴───────┐
   │ memberships │   │users │  │ retention_cases   │  │  retention_   │
   └─────────────┘   └──────┘  └────┬─────────────┘  │   actions     │
                                   │              └────┴───────┬───────┘
                        ┌──────────┼──────────┐                │
                        │          │          │                │
                 ┌──────▼───┐ ┌────▼─────┐ ┌──▼──────────────┐ │
                 │attendance│ │ messages │ │ risk_scores     │ │
                 └──────────┘ └────┬─────┘ └─────────────────┘ │
                                    │                           │
                             ┌──────▼──────┐                    │
                             │interactions │                    │
                             └──────┬──────┘                    │
                                    │                           │
                             ┌──────▼──────┐                    │
                             │ ai_analysis │────────────────────┘
                             └─────────────┘
```

### Tables

| Table | Purpose | Rows* |
|---|---|---|
| `gyms` | Tenant root | 1 |
| `users` | Gym staff and trainer login accounts | 2 |
| `trainers` | Coaching staff | 4 |
| `members` | Gym members — **the ownership anchor** | 4 |
| `memberships` | Subscription plans and periods | 4 |
| `attendance` | Session check-ins | 17 |
| `risk_scores` | Per-member risk history with sub-scores | 0 |
| `retention_cases` | **The unit of retention work** | 4 |
| `retention_actions` | Immutable audit trail | 78 |
| `retention_settings` | Per-gym thresholds and guardrails | 1 |
| `messages` | Outbound and inbound WhatsApp messages | 11 |
| `interactions` | Raw conversation turns | 18 |
| `ai_analysis` | Per-interaction AI classification | 0 |
| `notification_reads` | Which derived notifications a user has read | 3 |
| `password_reset_tokens` | Hashed, expiring reset tokens | 1 |
| `error_logs` | Automation failure log | 0 |

\* Live row counts at time of writing. Tiny because this is a single-gym system with
production-like data, not a seed dataset.

### Legacy tables

`_bak_members_20261008`, `_bak_retention_actions_20261008`, `_bak_retention_cases_20261008`,
`_bak_trainers_20261008`, `_bak_users_20261008` — pre-migration snapshots. Safe to drop
once the retention window has passed.

---

## 2. Table reference

### `gyms`

Tenant root. Every other table hangs off it.

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `name` | text | no | — |
| `timezone` | text | yes | `'UTC'` |
| `default_language` | text | yes | `'en'` |
| `avatar_url` | text | yes | — |
| `created_at` | timestamptz | yes | `now()` |

---

### `users`

Gym staff **and** trainer logins. One identity system — trainers log in through the same
Fastify + JWT + bcrypt stack as owners.

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `gym_id` | uuid | no | → `gyms.id` |
| `email` | text | no | — |
| `password_hash` | text | no | — |
| `first_name` | text | yes | — |
| `last_name` | text | yes | — |
| `role` | text | no | `'owner'` |
| `active` | boolean | no | `true` |
| `last_login_at` | timestamptz | yes | — |
| `avatar_url` | text | yes | — |
| `trainer_id` | uuid | yes | → `trainers.id` |
| `created_at` | timestamptz | yes | `now()` |
| `updated_at` | timestamptz | yes | `now()` |

**CHECK `role IN ('owner','manager','trainer')`**

**CHECK `users_trainer_role_ck`** — deny-by-default:
```sql
(role = 'trainer'  AND trainer_id IS NOT NULL) OR
(role <> 'trainer' AND trainer_id IS NULL)
```

**FK `(trainer_id, gym_id) → trainers (id, gym_id)`** — a trainer account cannot point at
a trainer in another gym. Requires the unique index `trainers_id_gym_unique ON (id, gym_id)`.

**UNIQUE** `(email)` · also `users_trainer_unique ON (trainer_id) WHERE trainer_id IS NOT NULL`
— one user per trainer.

**Indexes** `idx_users_email`, `idx_users_gym`, `idx_users_trainer_id`.

---

### `trainers`

Coaching staff. A trainer may exist with no login (see `has_account` in the API response).

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `gym_id` | uuid | no | → `gyms.id` |
| `name` | text | no | — |
| `phone` | text | no | — |
| `email` | text | yes | — |
| `active` | boolean | yes | `true` |
| `avatar_url` | text | yes | — |
| `created_at` | timestamptz | yes | `now()` |

**UNIQUE** `(gym_id, lower(email)) WHERE email IS NOT NULL` · `(id, gym_id)`

**Indexes** `idx_trainers_gym`, PK.

> `active = false` removes a trainer from reassignment targets and fails the workflow's
> `Trainer Authorized?` gate. It does not null their historical case records.

---

### `members`

**The ownership anchor.** `assigned_trainer_id` here is the single source of truth for who
owns any case belonging to this member.

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `gym_id` | uuid | no | → `gyms.id` |
| `first_name` | text | no | — |
| `last_name` | text | no | — |
| `phone` | text | no | — |
| `email` | text | yes | — |
| `preferred_language` | text | yes | `'en'` |
| `assigned_trainer_id` | uuid | yes | → `trainers.id` |
| `normal_sessions_per_week` | numeric(4,2) | yes | — |
| `contact_allowed` | boolean | yes | `true` |
| `status` | text | no | `'active'` |
| `created_at` | timestamptz | yes | `now()` |
| `updated_at` | timestamptz | yes | `now()` |

**CHECK** `status IN ('active','inactive')`
**CHECK** `normal_sessions_per_week IS NULL OR >= 0`

**Indexes** `idx_members_gym`, `idx_members_trainer`, PK.

> **`contact_allowed` is a consent switch.** Every automation entry query filters on it, so
> a member with it off is structurally excluded from outreach, follow-up and inbound
> matching — not merely skipped after selection.
>
> **`normal_sessions_per_week` is the baseline** attendance is measured against. A member
> who normally trains twice a week and skips once is behaving normally; the same skip is a
> decline for someone who trains five times.

---

### `memberships`

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `member_id` | uuid | no | → `members.id` |
| `start_date` | date | no | — |
| `end_date` | date | no | — |
| `status` | text | yes | `'active'` |
| `plan` | text | yes | — |
| `monthly_value` | numeric(10,2) | yes | — |
| `created_at` | timestamptz | yes | `now()` |

**CHECK** `status IN ('active','inactive','cancelled')`
**CHECK** `end_date >= start_date`
**CHECK** `monthly_value IS NULL OR >= 0`

**Indexes** `idx_membership_member`, `idx_memberships_end_date`,
`idx_memberships_member_dates`, `idx_memberships_member_status_end`,
`idx_memberships_status`.

> `monthly_value` is what makes revenue-at-risk computable.

---

### `attendance`

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `member_id` | uuid | no | → `members.id` |
| `session_date` | date | no | — |
| `session_type` | text | yes | — |
| `trainer_id` | uuid | yes | → `trainers.id` |
| `status` | text | yes | `'attended'` |
| `duration` | integer | yes | — |
| `created_at` | timestamptz | yes | `now()` |

**CHECK** `status IN ('attended','cancelled')`
**CHECK** `duration IS NULL OR >= 0`

**Indexes** `idx_attendance_date`, `idx_attendance_member_date`, PK.

---

### `risk_scores`

Per-member risk history with the four contributing sub-scores, kept for explainability.

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `member_id` | uuid | no | → `members.id` |
| `score` | numeric(5,2) | no | — |
| `level` | text | no | — |
| `attendance_score` | numeric(5,2) | yes | — |
| `trend_score` | numeric(5,2) | yes | — |
| `membership_score` | numeric(5,2) | yes | — |
| `history_score` | numeric(5,2) | yes | — |
| `calculated_at` | timestamptz | yes | `now()` |

**CHECK** `level IN ('healthy','watch','at_risk','high_risk','critical')`
**CHECK** `score BETWEEN 0 AND 100` and the same bound on each sub-score

**Indexes** `idx_risk_scores_member_date`, PK.

> **Currently empty (0 rows).** Retention Detection computes risk in Code nodes and writes
> `retention_cases`, but does not persist a `risk_scores` row. The table and its indexes
> exist and are ready; the insert is simply not wired up. Either wire it or drop it — an
> always-empty table is misleading.

---

### `retention_cases`

**The unit of retention work.**

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `member_id` | uuid | no | → `members.id` |
| `risk_score` | numeric(5,2) | no | `0` |
| `risk_level` | text | yes | `'healthy'` |
| `trigger` | text | no | — |
| `status` | text | yes | `'open'` |
| `reason` | text | yes | — |
| `assigned_trainer_id` | uuid | yes | → `trainers.id` |
| `created_at` | timestamptz | yes | `now()` |
| `updated_at` | timestamptz | yes | `now()` |
| `resolved_at` | timestamptz | yes | — |
| `snoozed_until` | timestamptz | yes | — |

**CHECK** `risk_level IN ('healthy','watch','at_risk','high_risk','critical')`
**CHECK** `risk_score BETWEEN 0 AND 100`
**CHECK** `status IN ('open','in_progress','resolved','snoozed','closed')`

**Indexes** `idx_retention_cases_member`, `idx_retention_cases_open`,
`idx_retention_cases_status`, PK.

#### Status semantics

| Status | Meaning | Reached by |
|---|---|---|
| `open` | Unworked | Detection; reassignment reopens |
| `in_progress` | A trainer or the automation has acted | `contact`; `snooze` |
| `snoozed` | Parked with no date | Raw status edit only — **not** the snooze action |
| `resolved` | Handled, member retained | `resolve` |
| `closed` | Handled, member lost | Staff action |

> **A snooze sets `in_progress`, not `snoozed`.** A snooze pauses outreach via
> `snoozed_until`; every workflow honours that date. Using `snoozed` instead would drop the
> case out of the trainer's reminder queue — removing it from the one process that stops a
> case being forgotten. The value remains legal in the schema for the raw status edit.

#### `assigned_trainer_id` is a cache

Do not write it. It is maintained by trigger. See [case ownership](#3-case-ownership).

---

### `retention_actions`

Immutable audit trail. One row per action, never updated.

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `case_id` | uuid | no | → `retention_cases.id` |
| `action_type` | text | no | — |
| `performed_by` | text | no | — |
| `performed_by_id` | uuid | yes | → `users.id` ON DELETE SET NULL |
| `performed_at` | timestamptz | yes | `now()` |
| `result` | text | yes | — |
| `notes` | text | yes | — |

**CHECK** `action_type IN (…)` — **13 distinct values** across 14 array entries:
`case_created`, `message_sent`, `member_replied`, `trainer_notified`, `trainer_contacted`,
`followup_scheduled`, `case_escalated`, `case_reopened`, `case_resolved`, `case_closed`,
`case_snoozed`, `trainer_reassigned`, `note_added`, `case_reopened` ← duplicate

> The array literal lists `case_reopened` twice. Harmless — the value is accepted once —
> but it is a defect in the constraint definition worth cleaning up.

**FK** `performed_by_id → users(id) ON DELETE SET NULL`

**Indexes** `idx_actions_case`, PK.

> `performed_by_id` is nullable with `ON DELETE SET NULL` so deleting a user does not
> cascade-delete history. The row survives; the actor link is severed. The actor's
> `performed_by` label remains as a human-readable record.

**Live distribution:** `message_sent` 34, `trainer_notified` 32, `case_resolved` 5,
`case_created` 4, `case_reopened` 2, `case_escalated` 1.

---

### `retention_settings`

Per-gym configuration. Exactly one row per gym.

| Column | Type | Null | Default | Live |
|---|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` | |
| `gym_id` | uuid | no | → `gyms.id` | |
| `inactivity_threshold_days` | integer | yes | `10` | **14** |
| `watch_threshold` | numeric(5,2) | yes | `30` | **30** |
| `at_risk_threshold` | numeric(5,2) | yes | `50` | **50** |
| `high_risk_threshold` | numeric(5,2) | yes | `70` | **70** |
| `critical_threshold` | numeric(5,2) | yes | `85` | **85** |
| `followup_delay_days` | integer | yes | `3` | **4** |
| `max_automated_outreach` | integer | yes | `3` | **3** |
| `cooldown_hours` | integer | yes | `48` | **48** |
| `updated_at` | timestamptz | yes | `now()` | |

**UNIQUE** `(gym_id)`

**CHECK thresholds are strictly increasing:**
```sql
watch_threshold < at_risk_threshold
  AND at_risk_threshold < high_risk_threshold
  AND high_risk_threshold < critical_threshold
```

**CHECK** each threshold between 0 and 100; all three integer settings `>= 0`.

> The ordering constraint is what makes a misconfiguration impossible. A gym cannot save
> `watch = 90` and `critical = 50`, which would make every member permanently critical.

---

### `messages`

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `member_id` | uuid | no | → `members.id` |
| `case_id` | uuid | yes | → `retention_cases.id` |
| `direction` | text | no | — |
| `channel` | text | yes | `'whatsapp'` |
| `message` | text | no | — |
| `provider_message_id` | text | yes | — |
| `status` | text | yes | `'pending'` |
| `sent_at` | timestamptz | yes | — |
| `delivered_at` | timestamptz | yes | — |
| `read_at` | timestamptz | yes | — |
| `created_at` | timestamptz | yes | `now()` |

**CHECK** `channel = 'whatsapp'`
**CHECK** `direction IN ('inbound','outbound')`
**CHECK** `status IN ('pending','sent','delivered','read','failed')`

**Indexes** `idx_messages_case`, `idx_messages_member`, `idx_messages_member_date`, PK.

> **`sent_at` is set when the provider accepts the message, not when it is read.**
> `delivered_at` and `read_at` exist but nothing currently populates them — Evolution API
> does not deliver those receipts into this table. Treat delivery as unverified.

> **The per-case outreach cap counts rows where** `case_id` matches **AND** `direction='outbound'`
> **AND** `channel='whatsapp'` **AND** `status IN ('sent','delivered','read')` **AND**
> `sent_at IS NOT NULL`. It is a **lifetime cap per case**, not per member per period — a
> member whose case is resolved and later re-flagged gets a fresh allowance. See
> [known issues](deployment.md#known-issues).

---

### `interactions`

Raw conversation turns. Distinct from `messages`: `messages` is what was sent and its
delivery state, `interactions` is the conversation log the AI reads.

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `member_id` | uuid | no | → `members.id` |
| `channel` | text | yes | `'whatsapp'` |
| `direction` | text | no | — |
| `message` | text | no | — |
| `created_at` | timestamptz | yes | `now()` |

**CHECK** `channel = 'whatsapp'`, `direction IN ('inbound','outbound')`

**Indexes** `idx_interactions_member_date`, PK.

---

### `ai_analysis`

Per-interaction AI classification.

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `interaction_id` | uuid | no | → `interactions.id` |
| `intent` | text | no | — |
| `confidence` | numeric(5,4) | no | — |
| `sentiment` | text | yes | — |
| `requires_human` | boolean | yes | `false` |
| `recommended_action` | text | yes | — |
| `reason` | text | yes | — |
| `raw_response` | jsonb | yes | — |
| `created_at` | timestamptz | yes | `now()` |

**CHECK** `confidence BETWEEN 0 AND 1`
**CHECK** `sentiment IS NULL OR IN ('positive','neutral','negative')`

**Indexes** PK.

> Currently empty. The Inbound Response Engine's `Parse AI Analysis` node classifies the
> reply to route the decision, but the `ai_analysis` row is not persisted — so there is no
> historical record of why the system escalated or auto-replied. This is a real gap for
> auditing AI behaviour; the table is ready for it.

---

### `notification_reads`

Notifications are **derived live** from existing data (open cases, inbound replies,
expiring memberships). This table persists only which ones a user has read, so read state
survives a reload without fabricating notification rows.

| Column | Type | Null | Default |
|---|---|---|---|
| `user_id` | uuid | no | → `users.id` ON DELETE CASCADE |
| `notification_id` | text | no | — |
| `read_at` | timestamptz | no | `now()` |

**PRIMARY KEY** `(user_id, notification_id)` · **Index** `notification_reads_user_idx`

---

### `password_reset_tokens`

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `user_id` | uuid | no | → `users.id` |
| `token_hash` | text | no | — |
| `expires_at` | timestamptz | no | — |
| `used_at` | timestamptz | yes | — |
| `created_at` | timestamptz | yes | `now()` |

**UNIQUE** `(token_hash)` — the raw token is never stored, only its hash.

**Indexes** `idx_password_reset_tokens_expires`, `idx_password_reset_tokens_user`, PK.

---

### `error_logs`

Automation failure log. Written by workflow error handlers.

| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | no | `gen_random_uuid()` |
| `workflow` | text | no | — |
| `node` | text | yes | — |
| `error_type` | text | yes | — |
| `message` | text | yes | — |
| `member_id` | uuid | yes | → `members.id` |
| `occurred_at` | timestamptz | yes | `now()` |
| `retry_count` | integer | yes | `0` |
| `resolved` | boolean | yes | `false` |

**CHECK** `retry_count >= 0`

**Indexes** `idx_error_logs_member`, `idx_errors_unresolved`, PK.

> Currently empty — and that is **not** evidence the automation is error-free. Failures have
> been observed in n8n execution history; they simply are not reaching this table.

---

## 3. Case ownership

Ownership follows the member. Enforced by two triggers, not by convention.

```
                    members.assigned_trainer_id          ← SOURCE OF TRUTH
                              │
          ┌───────────────────┴───────────────────┐
          │                                       │
  BEFORE UPDATE on                AFTER UPDATE on
  retention_cases                members
          │                                       │
          ▼                                       ▼
  overwrite any write to          copy the new owner to
  retention_cases                every case of this member
  .assigned_trainer_id            whose status is
  with the member's value         open/in_progress/snoozed
```

### `trg_retention_cases_inherit_trainer` — BEFORE INSERT OR UPDATE OF `member_id, assigned_trainer_id`

```sql
SELECT m.assigned_trainer_id INTO NEW.assigned_trainer_id
  FROM members m WHERE m.id = NEW.member_id;
```

Guarantee: a case can never hold a trainer different from its member's, by any code path.

### `trg_members_propagate_trainer` — AFTER UPDATE OF `assigned_trainer_id` on `members`

```sql
IF NEW.assigned_trainer_id IS DISTINCT FROM OLD.assigned_trainer_id THEN
  UPDATE retention_cases
     SET assigned_trainer_id = NEW.assigned_trainer_id, updated_at = now()
   WHERE member_id = NEW.id
     AND status IN ('open', 'in_progress', 'snoozed');
END IF;
```

Guarantee: reassigning a member moves their **active** work. Resolved and closed cases keep
the trainer who handled them — history is not rewritten.

### Consequence for the n8n workflows

**Writing `retention_cases.assigned_trainer_id` directly does nothing** — the BEFORE trigger
overwrites it from the member on the same statement. This is not a theoretical concern: the
`Reassign Case` node previously did exactly that, and reassignment was a silent no-op while
the ownership-drift metric read zero.

Reassignment must `UPDATE members`. The trigger propagates. The workflow has been changed
accordingly.

### Drift check

```sql
SELECT COUNT(*) FROM retention_cases rc
  JOIN members m ON m.id = rc.member_id
 WHERE rc.assigned_trainer_id IS DISTINCT FROM m.assigned_trainer_id;
```

**Must return 0.** Currently 0. `ensureSchema()` also reconciles drift at every API boot.

---

## 4. Triggers

| Trigger | Table | Timing | Function |
|---|---|---|---|
| `trg_members_updated_at` | `members` | BEFORE ROW | `set_updated_at()` |
| `trg_retention_cases_updated_at` | `retention_cases` | BEFORE ROW | `set_updated_at()` |
| `trg_retention_settings_updated_at` | `retention_settings` | BEFORE ROW | `set_updated_at()` |
| `trg_retention_cases_inherit_trainer` | `retention_cases` | BEFORE ROW | `retention_cases_inherit_trainer()` |
| `trg_members_propagate_trainer` | `members` | AFTER ROW | `members_propagate_trainer_to_cases()` |

The application only has three functions of its own; everything else in `pg_proc` belongs to
the `pgvector` extension.

---

## 5. Schema management

### Boot-time guard

`ensureSchema()` runs on every API start and is idempotent — `ADD COLUMN IF NOT EXISTS`,
`CREATE INDEX IF NOT EXISTS`, and `DROP CONSTRAINT IF EXISTS` before `CREATE CONSTRAINT`
(Postgres has no `ADD CONSTRAINT IF NOT EXISTS`). It also reconciles any ownership drift
that predates the triggers.

### Migration files

`backend/src/db/migrations/` — the same changes, applied by hand or by a runner:

| File | Contents |
|---|---|
| `001_notifications.sql` | `notification_reads` table and index |
| `002_trainer_accounts.sql` | Trainer logins on the existing auth system; `users.trainer_id`, role CHECK, composite gym FK |
| `003_case_ownership_follows_member.sql` | The ownership model — both triggers plus a one-time drift reconciliation |

All three are written to be safe to run more than once.

> `002` creates a `trainer_accounts` table that `ensureSchema()` then drops
> (`DROP TABLE IF EXISTS trainer_accounts`) — trainer accounts were ultimately modelled as
> ordinary `users` rows with a `trainer_id` pointer. The table is gone; the comment in
> `ensure-schema.ts` explains why.

### Rules for changing this schema

- **Validate every statement with `PREPARE` before applying it**, inside a rolled-back
  transaction.
- **Do not write n8n state through SQL.** Change workflows with
  `export:workflow` → `import:workflow` → `publish:workflow` → `docker restart n8n`.
- Note that `export:workflow` returns the **active** version, not the draft. Round-tripping
  an export after editing in the UI will silently discard the draft.
- **Adding an invariant?** Prefer a CHECK constraint over application validation. It is
  enforced regardless of which code path writes.

---

## 6. Operational queries

```sql
-- Ownership drift. MUST be 0.
SELECT COUNT(*) FROM retention_cases rc
  JOIN members m ON m.id = rc.member_id
 WHERE rc.assigned_trainer_id IS DISTINCT FROM m.assigned_trainer_id;

-- Cases per status.
SELECT status, COUNT(*) FROM retention_cases GROUP BY status ORDER BY 2 DESC;

-- Open case load per trainer.
SELECT t.name, COUNT(*) FILTER (WHERE rc.status IN ('open','in_progress')) AS active,
       COUNT(*) FILTER (WHERE rc.status = 'snoozed') AS snoozed
  FROM retention_cases rc JOIN trainers t ON t.id = rc.assigned_trainer_id
 GROUP BY t.name ORDER BY 2 DESC;

-- Automated outreach per case (the cap is 3).
SELECT case_id, COUNT(*) FROM messages
 WHERE direction = 'outbound' AND channel = 'whatsapp'
   AND status IN ('sent','delivered','read') AND sent_at IS NOT NULL
 GROUP BY case_id HAVING COUNT(*) >= 3;

-- Full audit trail for one case.
SELECT performed_at, action_type, performed_by, performed_by_id, notes
  FROM retention_actions WHERE case_id = $1 ORDER BY performed_at;

-- Members who must never be messaged.
SELECT id, first_name, last_name FROM members WHERE contact_allowed = false;
```