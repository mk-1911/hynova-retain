# n8n Workflows

Six active automation workflows. Four run on a 5-minute cycle; two are webhooks.

Extracted from the live n8n instance on 2026-10-13 (n8n 2.31.7).

---

## 1. Inventory

| Workflow | ID | Active version | Nodes | Trigger |
|---|---|---|---|---|
| Retention Detection | `stYSbra8ljdaZHFZ` | `c9b82226` | 20 | Schedule, 5 min |
| Retention Outreach | `BK7ReBtVG4OnQ3xY` | `39f9aac6` | 23 | Schedule, 5 min |
| Follow-up & Case Resolution | `RMUBRxwBkRZ0VOf5` | `32988c45` | 22 | Schedule, 5 min |
| Trainer Workflow | `wRVVD3n8fARwWUVo` | `e2b13dd5` | 8 | Schedule, 5 min |
| Inbound Response Engine | `n79KfQzcqeNAZnLQ` | `be240cd1` | 25 | Webhook |
| Case Resolution | `C93GjaGts8Ng1wD6` | `4202bda7` | 19 | Webhook |

Seven further workflows exist in the instance and are **inactive and unrelated** — six
`CE-*` content-marketing workflows and one archived `Gym Retention System` (`7QGMiTqM7UtZ6CO6`).
They do not touch the application database.

### The schedule

All four scheduled workflows store their rule as:

```json
{ "rule": { "interval": [ { "field": "minutes" } ] } }
```

**The interval value is missing from the stored rule.** The cadence is nevertheless
unambiguous: execution timestamps align exactly 5 minutes apart
(`04:30:22`, `04:35:22`, `04:40:22` …), so n8n is defaulting to 5. The missing value is a
latent trap — opening any of these workflows and saving it from the editor could bake in a
different interval. **Set the value explicitly when next editing a schedule.**

Observed cadence, Retention Detection, most recent runs:

```
05:20:22  gap= 5.0m
05:15:22  gap= 5.0m
05:10:22  gap= 5.0m
05:05:22  gap= 5.0m
05:00:22  gap= 5.0m
...
```

> Larger gaps (a 37-hour one is in the history) are the host being off, not the schedule
> misbehaving. The system is not resilient to the machine sleeping — see
> [deployment.md](deployment.md#known-issues).

---

## 2. Retention Detection

`stYSbra8ljdaZHFZ` · `c9b82226` · 20 nodes · schedule

The only workflow that creates retention cases.

```
Schedule Trigger (5 min)
  │
  ├─ Get Active Gyms
  ├─ Get Active Members
  ├─ Get Active Memberships
  ├─ Get Attendance Metrics
  │
  ├─ Calculate Attendance Score    ┐
  ├─ Calculate Attendance Trend    │
  ├─ Get Membership Risk           │  Code nodes,
  ├─ Calculate Membership Risk     │  runOnceForEachItem
  ├─ Get Retention History         │
  ├─ Calculate History Risk        │
  ├─ Calculate Final Risk Score    ┘
  │
  ├─ Get Retention Settings        per-gym thresholds
  ├─ Determine Risk Level
  ├─ Retention Eligible?
  ├─ Check Existing Retention Case?
  │
  ├─ Create Retention Case         INSERT retention_cases
  └─ Record Case Creation          INSERT retention_actions (case_created)
```

**Inputs:** active gyms → active members with `contact_allowed` → their active memberships
→ last 30 days of attendance.

**Scoring:** four sub-scores (attendance, trend, membership, history) combined into a
0–100 composite, then banded by the gym's `retention_settings` thresholds.

**Idempotency:** `Check Existing Retention Case?` runs every cycle, so a member already
carrying an open case is skipped. A member cannot accumulate duplicate cases for one
episode.

> Every Code node here uses `runOnceForEachItem`. An earlier revision used
> `runOnceForAllItems` and returned a single-element array, which silently collapsed the
> batch — found by reading actual execution data, not by inspection.

> `risk_scores` is **not** written by this workflow. The risk calculation exists only in
> Code nodes and lands in `retention_cases`. See
> [database.md § risk_scores](database.md#riskscores).

---

## 3. Retention Outreach

`BK7ReBtVG4OnQ3xY` · `39f9aac6` · 23 nodes · schedule

Sends at most one AI-written WhatsApp message per case per cycle.

```
Schedule Trigger (5 min)
  │
  ├─ Get Eligible Retention Cases   status='open' AND contact_allowed = true
  ├─ Load Member + Trainer Context
  ├─ Get Retention Settings
  │
  ├─ Check Outreach Limits           per-case lifetime cap
  ├─ Check Outreach Cooldown         last sent_at on this case
  ├─ Normalize Cooldown              runOnceForEachItem
  ├─ Cooldown Passed?
  ├─ Outreach Allowed?
  ├─ Check Outreach Eligibility      final gate
  │
  ├─ Determine Outreach Action
  ├─ Need Trainer Escalation?
  │    └─ Send Trainer Alert
  │    └─ Log Trainer Escalation
  │
  ├─ Generate Retention Message     AI (Groq, qwen3.8-27b)
  ├─ Create Message Record           INSERT messages (pending)
  ├─ Send WhatsApp Message           POST evolution_api
  ├─ WhatsApp Send Successful?
  │    ├─ Update Message Status       → sent
  │    └─ Handle Send Failure
  │         └─ Update Failed Message Status → failed
  └─ Log Retention Action            INSERT retention_actions (message_sent)
```

### The two guardrails

Both are enforced in SQL, not in application code.

**Cooldown** — `Normalize Cooldown` → `Cooldown Passed?`. Previously this pair was
unsatisfiable (the condition required `>` **and** `=`), so it either always passed or never
did. The cooldown was also computed but never consumed, so a member could be messaged every
5 minutes indefinitely.

**Cap** — `Check Outreach Limits` counts, per case:

```sql
WHERE case_id = $1
  AND direction = 'outbound'
  AND channel = 'whatsapp'
  AND status IN ('sent','delivered','read')
  AND sent_at IS NOT NULL
```

A **lifetime cap per case**. Not per member, not per period — a member whose case is
resolved and later re-flagged receives a fresh allowance. This is a known, deliberate
limitation; see [deployment.md](deployment.md#known-issues).

> `Check Outreach Cooldown` originally lacked its `LEFT JOIN`, returning 0 rows whenever
> there was no prior message — which deleted the item from the workflow rather than letting
> it through.

### Verified behaviour

- One cycle → one message, `status = 'sent'`, `sent_at` set, provider message id recorded,
  `message_sent` action logged.
- Second cycle inside the cooldown → suppressed. No duplicate row, no duplicate action.
- Member re-armed (`contact_allowed` on, cooldown cleared) → messages again.

---

## 4. Follow-up & Case Resolution

`RMUBRxwBkRZ0VOf5` · `32988c45` · 22 nodes · schedule

Follows up on ageing cases and escalates when the automation is not resolving them.

```
Schedule Trigger (5 min)
  │
  ├─ Get Cases Needing Follow-up
  ├─ Load Case + Member Context
  ├─ Determine Follow-up State      runOnceForEachItem
  ├─ Follow-up Required?
  ├─ Determine Follow-up Action
  ├─ Follow-up Action                switch
  ├─ Check Follow-up Eligibilty
  ├─ Still needs Follow-up?
  │
  ├─ Check Assigned Trainer
  ├─ Assigned Trainer Exists?
  │    ├─ Send Trainer Alert
  │    │    └─ Log Trainer Escalation
  │    └─ Log Unassigned Trainer Escalation
  │
  ├─ Generate Member Response        AI
  ├─ Create Message Record
  ├─ Send WhatsApp Message
  ├─ WhatsApp Send Successful?
  ├─ Update Message Status
  └─ Log Retention Action
```

Triggers on `followup_delay_days` (live value: 4). Escalates rather than messaging when no
trainer is assigned — an unassigned case is a staffing problem, not a messaging problem.

---

## 5. Trainer Workflow

`wRVVD3n8fARwWUVo` · `e2b13dd5` · 8 nodes · schedule

The accountability loop. Reminds a trainer about cases they have not actioned.

```
Schedule Trigger (5 min)
  │
  ├─ Get Open Retention Cases
  ├─ Load Case + Member + Trainer
  ├─ Get Trainer Case Action        alwaysOutputData
  ├─ Determine Trainer Follow-up State
  ├─ Trainer Action Required
  ├─ Send Trainer Reminder
  └─ Log Trainer Reminder
```

### The multi-case crash

This workflow originally crashed whenever a trainer had more than one case. One case
succeeded; four produced:

```
ExpressionError: Multiple matches
pairedItemMultipleMatches
```

The cause was an expression resolving against a multi-item node without `itemIndex`
disambiguation. Fixed by loading case context through a CTE plus `LEFT JOIN` so each item
carries its own `case_id`, and grouping the Code node by `case_id`.

**Verified:** 53 open cases → 4 items, 3 cases reminded once, and the next cycle
deduplicated them.

> `Get Trainer Case Action` originally filtered `performed_by_id = <user id>`, but rows are
> written with `performed_by_id = NULL`, so it matched 0 of 50 rows and every trainer was
> re-reminded about everything, forever.

---

## 6. Inbound Response Engine

`n79KfQzcqeNAZnLQ` · `be240cd1` · 25 nodes · webhook

Handles member replies.

### Endpoint

```
POST http://localhost:5678/webhook/hynova-retain/inbound
```

| Property | Value |
|---|---|
| Method | `POST` |
| Path | `/webhook/hynova-retain/inbound` |
| Response mode | last node |
| **Authentication** | **none** |

### Flow

```
Inbound WhatsApp Webhook
  │
  ├─ Identify Member            by phone
  ├─ Find Active Retention Case
  ├─ Reserve Interaction        deduplication guard, runOnceForEachItem
  ├─ Save Interaction           INSERT interactions
  │
  ├─ AI Analyze Reply           intent / sentiment / confidence / requires_human
  ├─ Parse AI Analysis          runOnceForEachItem
  ├─ Determine Response Action
  │
  ├── requires_human ──▶ Get Trainer Details
  │                     ├─ Assigned Trainer Exists?
  │                     │    ├─ Send Trainer Alert → Log Trainer Escalation
  │                     │    └─ Log Unassigned Trainer Escalation
  │
  └── else ──▶ Generate Member Response   (AI)
               Generate Confirmation       (AI, when escalating)
               Create Message Record
               Save Message
               Send WhatsApp Message
               WhatsApp Send Successful?
               Update Message Status
               Log Retention Action
```

`requires_human` is set for injury, health concerns, serious complaints, strong
dissatisfaction, cancellation intent, or anything unclear.

### Known non-determinism

`Identify Member` matches on phone with `LIMIT 1`. **When two members share a phone number —
which the test data does — the reply is attributed to whichever the planner returns first.**
This is not deterministic across runs and is a correctness issue, not a cosmetic one. See
[known issues](deployment.md#known-issues).

> Do not test this endpoint without first confirming that `Identify Member` resolves to the
> intended QA member.

---

## 7. Case Resolution

`C93GjaGts8Ng1wD6` · `4202bda7` · 19 nodes · webhook

Trainer case actions. Called by the Fastify API — **never by the browser**.

### Endpoint

```
POST http://localhost:5678/webhook/gym-retention-case-action
```

| Property | Value |
|---|---|
| Method | `POST` |
| Path | `/webhook/gym-retention-case-action` |
| Response mode | `responseNode` |
| **Authentication** | **none** |

### Request

```json
{
  "case_id":     "uuid",
  "trainer_id":  "uuid",
  "action":      "contact | resolve | snooze | reassign",
  "note":        "optional string",
  "snooze_days": 1,                    // required for snooze
  "new_trainer_id": "uuid"             // required for reassign
}
```

`trainer_id` is supplied by the API from the verified token. It is never accepted from a
browser.

### Flow

```
Case Action Webhook
  │
  ├─ Validate Case Action          onError = continueErrorOutput
  │    └─ Invalid Request  → 400
  │
  ├─ Load Case + Trainer           alwaysOutputData
  ├─ Trainer Authorized?           assigned_trainer_id === trainer_id AND trainer.active
  │    └─ Not Authorized → 403
  │
  ├─ Determine Action              switch
  │    ├─ contact  → Mark Trainer Contacted → Log Trainer Contact
  │    ├─ resolve  → Resolve Case          → Log Case Resolution
  │    ├─ snooze   → Snooze Case           → Log Case Snooze
  │    └─ reassign → Validate New Trainer → New Trainer Valid?
  │                    └─ Invalid → 400
  │                 → Reassign Case        → Log Trainer Reassignment
  │
  └─ Success → 200
```

### Responses

| Node | Status | Body |
|---|---|---|
| `Success` | 200 | `{ success: true, message, case_id, action }` |
| `Not Authorized` | 403 | `{ success: false, error: "Trainer is not authorized to modify this case." }` |
| `Invalid` | 400 | `{ success: false, error: "New trainer is invalid, inactive, or does not belong to this gym." }` |
| `Invalid Request` | 400 | `{ success: false, error: "Invalid request. case_id, trainer_id and action (…) are required." }` |

### Action semantics

| Action | Mutation | Audit row | Case status after |
|---|---|---|---|
| `contact` | `status = 'in_progress'` | `trainer_contacted` | `in_progress` |
| `resolve` | `status = 'resolved'`, `resolved_at = now()` | `case_resolved` | `resolved` |
| `snooze` | `status = 'in_progress'`, `snoozed_until = now() + snooze_days` | `case_snoozed` | `in_progress` |
| `reassign` | `UPDATE members SET assigned_trainer_id` | `trainer_reassigned` | `open` |

### Reassignment writes the member

`Reassign Case` updates **`members.assigned_trainer_id`**, never the case column directly:

```sql
WITH target AS (
    SELECT rc.id AS case_id, rc.member_id
      FROM retention_cases rc
     WHERE rc.id = $1
       AND rc.status IN ('open', 'in_progress')
),
moved AS (
    UPDATE members m
       SET assigned_trainer_id = $2, updated_at = NOW()
      FROM target t
     WHERE m.id = t.member_id
    RETURNING m.id AS member_id, m.assigned_trainer_id AS new_trainer_id
)
UPDATE retention_cases rc
   SET status = 'open', snoozed_until = NULL, updated_at = NOW()
  FROM target t, moved
 WHERE rc.id = t.case_id
RETURNING rc.id AS case_id, moved.new_trainer_id AS assigned_trainer_id,
          rc.status, rc.updated_at;
```

The `members` UPDATE moves ownership; `trg_members_propagate_trainer` syncs the case. The
second statement reopens the case and clears any snooze, but **never** sets
`assigned_trainer_id` — that is the trigger's job, and writing it here would be reverted
anyway by `trg_retention_cases_inherit_trainer`.

> The previous implementation wrote the case column directly. Because the BEFORE trigger
> overwrites it from the member on the same statement, **reassignment was a silent no-op.**

### `Validate New Trainer` checks the target

It previously validated the **acting** trainer against the case's gym, so the reassignment
target was never checked — an inactive trainer or one from another gym passed silently. It
now validates the target:

```sql
SELECT t.id AS trainer_id, t.gym_id, t.name AS trainer_name, …
  FROM trainers t
  JOIN members m      ON m.gym_id = t.gym_id
  JOIN retention_cases rc ON rc.member_id = m.id
 WHERE t.id = $1                      -- new_trainer_id
   AND rc.id = $2
   AND t.active = true
 LIMIT 1;
```

**Guard matrix (all pass):**

| Condition | Rows |
|---|---|
| Active trainer, same gym | 1 |
| Inactive trainer, same gym | 0 |
| Active trainer, **different** gym | 0 |
| Non-existent trainer id | 0 |

---

## 8. Patterns used across all workflows

### Query binding

Postgres nodes use `queryReplacement` with `$1, $2` positional binds. Member names and case
reasons reach the database as parameters, never as concatenated SQL.

### `runOnceForEachItem` on Code nodes

Any node that mutates or derives per-item state runs once per item. `runOnceForAllItems`
returned a single-element array and collapsed the batch — this caused a real, silent data
loss bug.

### `alwaysOutputData` on context lookups

`Load Case + Trainer Context`, `Get Retention Settings`, `Check Existing Retention Case`
and `Get Trainer Case Action` all set `alwaysOutputData` so a zero-row result does not
terminate the branch.

### Zero rows means no response

An `UPDATE` that matches nothing produces no items, so the downstream audit insert never
runs and no `respondToWebhook` node is reached. The webhook simply does not reply. The
backend proxy treats that as a timeout and surfaces a failure — it never reports success it
cannot substantiate. The proxy therefore pre-checks case state for `snooze` and `reassign`
and returns `409` before calling n8n when the case is already resolved or closed.

### WhatsApp delivery

```
POST http://evolution_api:8080/message/sendText/hynova-retain
```

Instance name `hynova-retain`. The hostname is a Docker-internal service name on the
`gym-net` network — the browser never contacts Evolution API directly.

---

## 9. Credentials

| Credential | Type | Used by |
|---|---|---|
| `Postgres account` | postgres | Application database |
| `Postgres account 2` | postgres | Second database connection |
| `Groq account` | groqApi | All 5 AI agent nodes |
| `Supabase account` | supabaseApi | — |
| `OpenAI account` | openAiApi | — |
| `Hugging Face account` | huggingFaceApi | — |
| `Qwen Cloud account` | alibabaCloudApi | — |

Only the Postgres and Groq credentials are in use. The remainder are unused and can be
deleted from the n8n instance.

> Credential **names** are listed; credential **values** are not, and should not be
> exported into documentation. An n8n API token is currently committed in plaintext at
> `n8n/eyJhbGciOi….txt` (a JWT-named file) — rotate and remove it. See
> [deployment.md](deployment.md#credential-hygiene).

---

## 10. Changing a workflow

**Never write n8n state through SQL.** n8n maintains its own cache and a direct row update
will be overwritten or leave the in-memory and stored versions disagreeing.

Use the supported cycle:

```bash
# 1. export the ACTIVE version
docker exec n8n n8n export:workflow --id=<WORKFLOW_ID> --pretty --output=/tmp/wf.json
docker cp n8n:/tmp/wf.json ./wf.json

# 2. edit ./wf.json

# 3. validate every SQL statement against the real database
#    PREPARE inside a transaction you then ROLL BACK

# 4. import and publish
docker cp ./wf.json n8n:/tmp/wf.json
docker exec n8n n8n import:workflow --input=/tmp/wf.json
docker exec n8n n8n publish:workflow --id=<WORKFLOW_ID>

# 5. restart
docker restart n8n

# 6. VERIFY THE PUBLISHED VERSION, not the draft
docker exec n8n-postgres psql -U n8n -d n8n -t -A -c \
  "SELECT nodes::text FROM workflow_history
    WHERE \"workflowId\"='<ID>' AND \"versionId\"=(SELECT \"activeVersionId\" FROM workflow_entity WHERE id='<ID>');"
```

### Two traps

1. **`export:workflow` returns the active version, not the draft.** Editing in the n8n
   editor and then round-tripping an export discards the draft silently.
2. **n8n executes the row named by `activeVersionId`.** A draft that is not published has no
   effect at all. This was the root cause of an extended period where fixes appeared to be
   applied but were not.

### Verify after publishing

Confirm the published version's node parameters directly, and confirm the `activeVersionId`
actually changed:

```bash
docker exec n8n-postgres psql -U n8n -d n8n -t -A -c \
  "SELECT name, \"activeVersionId\" FROM workflow_entity;"
```