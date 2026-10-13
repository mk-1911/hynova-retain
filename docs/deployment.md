# Deployment & Operations

Setup, configuration, runbook and known issues.

---

## 1. Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Node.js | 20+ (24 in use) | ESM throughout; `"type": "module"` |
| Docker Desktop | Current | Runs the whole automation stack |
| PostgreSQL | 15+ | `pgvector` extension enabled |
| WhatsApp Business account | — | Connected to the Evolution API instance |
| Groq API key | — | For the AI layer |

The application database is currently **Supabase** (remote, `eu-west-1` pooler) reached
over the internet. This works, but it means every automation cycle opens a WAN connection —
see [known issues](#known-issues).

---

## 2. Ports

| Port | Service | Bind | Exposed to browser? |
|---|---|---|---|
| 3000 | Fastify API | host | ✅ yes |
| 5173 | Vite dev server | host | ✅ yes |
| 5678 | n8n | host | ⚠️ should not be in production |
| 8080 | Evolution API | host | ❌ never |

**Evolution API must never be reachable from the browser.** It holds the WhatsApp session.
It is bound to the host here for local convenience; in production it should stay on the
Docker network only.

---

## 3. Docker stack

```bash
cd n8n
docker compose up -d
docker compose ps
```

Three services on a `gym-net` bridge network:

| Service | Container | Image |
|---|---|---|
| `postgres` | `n8n-postgres` | `postgres:16` |
| `n8n` | `n8n` | `n8nio/n8n:latest` |
| `evolution-api` | `evolution_api` | `evoapicloud/evolution-api:latest` |

**n8n stores its own state in `n8n-postgres`.** Evolution API shares the same server under
its own `evolution` schema, so there is one database volume to back up for the automation
layer.

Configuration highlights:

```yaml
N8N_HOST: localhost
N8N_PORT: 5678
N8N_PROTOCOL: http          # ⚠️ HTTP, no TLS — fine on localhost, not in production
GENERIC_TIMEZONE: Asia/Riyadh
TZ: Asia/Riyadh
NODE_TLS_REJECT_UNAUTHORIZED: "0"   # ⚠️ disables certificate checking, process-wide
NODE_OPTIONS: "--dns-result-order=ipv4first"
```

> `NODE_TLS_REJECT_UNAUTHORIZED: "0"` disables TLS certificate validation for **every**
> outbound request the container makes. It is there because of self-signed certificates on
> the Supabase connection. Remove it and supply a proper CA certificate instead — a blanket
> TLS bypass is not acceptable for a production deployment.

### Evolution API

- Instance name: **`hynova-retain`** — every message send uses this path
  `/message/sendText/hynova-retain`
- Auth type: `apikey`
- Persistence enabled: instance, new messages, message updates
- Cache: local only (Redis disabled)

### Operations

```bash
docker compose ps                 # status
docker compose logs -f n8n        # n8n log
docker compose restart n8n        # restart after publishing a workflow
docker compose down               # stop
docker compose down -v            # ⚠️ DESTROYS the database volume
```

---

## 4. Backend

### Environment variables

`backend/.env`:

| Variable | Required | Purpose |
|---|---|---|
| `DB_HOST` | ✅ | Supabase host, e.g. `aws-1-eu-west-1.pooler.supabase.com` |
| `DB_PORT` | ✅ | `5432` |
| `DB_USER` | ✅ | Database user |
| `DB_PASSWORD` | ✅ | Database password |
| `DB_NAME` | ✅ | Database name |
| `JWT_SECRET` | ✅ | Token signing secret. Rotating it invalidates all sessions. |
| `FRONTEND_URL` | ✅ | CORS origin. Exactly one origin. |
| `N8N_CASE_ACTION_WEBHOOK_URL` | ✅¹ | Case Action webhook **production** URL |
| `EMAIL_PROVIDER` | ➖ | `brevo` or `resend` |
| `EMAIL_FROM` | ➖ | Sender address |
| `BREVO_API_KEY` | ➖ | When provider is `brevo` |
| `RESEND_API_KEY` | ➖ | When provider is `resend` |
| `HEALTHCHECK_SECRET` | ➖ | Protects the health endpoint |
| `PORT` | ➖ | Default `3000` |
| `LOG_LEVEL` | ➖ | Default `info` |
| `TRUST_PROXY` | ➖ | `true` behind a reverse proxy |
| `NODE_ENV` | ➖ | `production` disables dev email behaviour |

¹ Required for the trainer case-action path. Without it the endpoint returns `503`.

```bash
cd backend
npm install
npm run dev     # tsx watch src/server.ts
npm start       # tsx src/server.ts
```

### Schema

**Applied automatically at boot.** `ensureSchema()` runs on start and is idempotent:
`ADD COLUMN IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, and `DROP CONSTRAINT IF EXISTS`
before `CREATE CONSTRAINT`. It also reconciles any ownership drift that predates the
triggers.

A fresh deployment needs no separate migration step. The SQL in `backend/src/db/migrations/`
contains the same changes for manual application or a migration runner.

Verify after first start:

```sql
SELECT COUNT(*) FROM retention_cases rc
  JOIN members m ON m.id = rc.member_id
 WHERE rc.assigned_trainer_id IS DISTINCT FROM m.assigned_trainer_id;
-- must be 0
```

---

## 5. Frontend

```bash
cd frontend
npm install
npm run dev       # vite, http://localhost:5173
npm run build     # tsc -b && vite build
npm run lint      # eslint
npm run preview   # serve the production build
```

The API base URL is not set in the frontend — the SPA calls the same origin and is expected
to sit behind a reverse proxy that routes `/api/*` (or equivalent) to port 3000.

Production bundle: **~293 kB, ~89 kB gzipped**, route-level code splitting on every page.

---

## 6. Roles and first run

Two account types, both ordinary `users` rows.

### Gym owner

Create a `users` row with `role = 'owner'` and a bcrypt hash of a password. A `CHECK`
constraint enforces that a non-trainer role must have `trainer_id IS NULL`.

### Trainer

1. Create the `trainers` row first.
2. `POST /trainers/:trainerId/invite` — generates an activation token and emails it.
3. The trainer opens `/trainer/activate`, sets a password.
4. A `users` row is created with `role = 'trainer'` and `trainer_id` set.

The composite FK `(trainer_id, gym_id) → trainers (id, gym_id)` prevents linking a trainer
account to a trainer in another gym.

---

## 7. Operations runbook

### Daily

```sql
-- Automation health
SELECT status, COUNT(*) FROM retention_cases GROUP BY status;
SELECT COUNT(*) FROM messages WHERE status = 'failed';
SELECT * FROM error_logs WHERE NOT resolved ORDER BY occurred_at DESC;

-- Outreach volume (should be small and steady)
SELECT DATE(sent_at), COUNT(*) FROM messages
 WHERE direction='outbound' AND sent_at > NOW() - INTERVAL '7 days'
 GROUP BY 1 ORDER BY 1;
```

In n8n: check that the four scheduled workflows ran on their 5-minute cadence and that
execution counts are not dropping to zero.

### Adding a member

`contact_allowed` defaults to `true`. For a member who must never be messaged, set it to
`false` — every automation entry query excludes them structurally.

### Reassigning a member

Update `members.assigned_trainer_id`. `trg_members_propagate_trainer` moves every case with
status `open`, `in_progress` or `snoozed`. Resolved and closed cases are left alone
intentionally.

### Reassigning a case

**Do not update `retention_cases.assigned_trainer_id`.** It is a trigger-maintained cache and
a direct write is overwritten on the same statement. Use the application, which goes through
the Case Resolution webhook.

### Tuning outreach volume

Tighten `cooldown_hours` and `max_automated_outreach` in `retention_settings`, or via
**Settings → Automation**. These are the two settings that govern brand risk. The threshold
ordering constraint prevents an invalid configuration.

---

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `503 Case automation is not configured` | `N8N_CASE_ACTION_WEBHOOK_URL` unset | Add it to `backend/.env` and **restart the API** — `tsx watch` reloads code but not `.env` |
| `503 Case automation is misconfigured` | URL contains `/webhook-test/` | Use the production webhook URL |
| `502 The automation workflow could not be reached` | n8n down or unreachable | `docker compose ps`; check port 5678 |
| `504 …did not respond in time` | Workflow exceeded 15 s, or matched no rows | Check n8n execution history; if the case is resolved, the action is a 409 from the API instead |
| `404` on a trainer's case | Case outside their scope — correct behaviour | Expected. Not a bug. |
| `409 already resolved or closed` | Snooze/reassign on a resolved case | Reopen the case first |
| No messages being sent | Cooldown, cap, or `contact_allowed` | Run the [operational query](database.md#operational-queries) for cases at the cap |
| Duplicate WhatsApp messages | Old workflow version still published | Check `activeVersionId` actually changed after publishing |
| n8n changes appear to do nothing | Draft edited but not published | n8n runs the `activeVersionId` row, not the draft |

### Reading n8n state

```bash
# active version per workflow
docker exec n8n-postgres psql -U n8n -d n8n -t -A -c \
  "SELECT name, \"activeVersionId\", active FROM workflow_entity ORDER BY name;"

# the node graph actually being executed
docker exec n8n-postgres psql -U n8n -d n8n -t -A -c \
  "SELECT nodes::text FROM workflow_history
    WHERE \"workflowId\"='<ID>' AND \"versionId\"=(SELECT \"activeVersionId\"
                                               FROM workflow_entity WHERE id='<ID>');"
```

---

## 9. Credential hygiene

> ⚠️ **Live credentials are committed to this repository in plaintext.** Rotate them and
> remove the files before this goes anywhere near a customer or a public repository.

| Location | Contents |
|---|---|
| `Supabase/httpskkuyrpzwxeehcyfnsvpv.supabase.corestv1.txt` | Database connection string with password |
| `Supabase/sb_secret_0IpIat…txt` | Supabase secret API key (full name is `sb_secret_….txt`) |
| `n8n/eyJhbGciOiJIUzI1NiIs…txt` | n8n API JWT |
| `n8n/docker-compose.yml` | `POSTGRES_PASSWORD`, `AUTHENTICATION_API_KEY` hardcoded |
| `backend/.env` | DB password, `JWT_SECRET`, Brevo/Resend keys — present in the working tree |

### Remediation

1. **Rotate everything**: Supabase database password, Supabase secret key, n8n API token,
   Brevo/Resend keys, and the n8n/Evolution Postgres password.
2. **Delete the credential files.**
3. **Add a `.gitignore`** covering `.env`, `*.txt` credentials, and `node_modules`.
4. **Move `docker-compose.yml` secrets to an env file** with `env_file: .env`.
5. **Generate `JWT_SECRET` fresh** if it has ever been shared.
6. If this repository has ever been pushed anywhere public, treat every value as
   compromised — rotation is the only remedy.

### Also required before production

- **Webhook authentication.** Both webhooks are open. Add header auth on the nodes and have
  the API send the header.
- **Remove `NODE_TLS_REJECT_UNAUTHORIZED: "0"`** and supply a CA bundle.
- **Terminate TLS.** n8n and Evolution API are on plain HTTP.
- **Restrict ports.** 5678 and 8080 should not be publicly bound.
- **Rotate `HEALTHCHECK_SECRET`** and confirm the health endpoint is not public.

---

## 10. Known issues

Ordered by severity. Every item has been observed in this deployment, not hypothesised.

### 1. Webhooks are unauthenticated — **critical**

Both `POST /webhook/gym-retention-case-action` and `POST /webhook/hynova-retain/inbound`
have `authentication: none`. Anyone who can reach port 5678 can:

- inject a fake member reply and trigger automated responses or escalations
- perform any of the four case actions as any trainer id, provided they know the case and
  trainer UUIDs

The backend proxy is the intended single caller, but nothing enforces that. **Fix before any
external customer connects**: set a header credential on both webhook nodes, and send it
from the API. The proxy already refuses `/webhook-test/` URLs.

### 2. No retry on Postgres nodes — **high**

**0 of 25 Postgres nodes set `retryOnFail`.** The application database is remote Supabase
over WAN. During verification, 29 connectivity failures were observed, and each silently
dropped that cycle's outreach or case creation.

Set `retryOnFail: true` with `maxTries` and `waitBetweenTries` on all 25 nodes. This is the
single highest-impact engineering fix available.

### 3. Host sleep stops all automation — **high**

The system runs on a machine that sleeps. Observed gaps include **37 hours** with no
executions. Nothing alerts, and the workflows only resume when the host wakes.

An always-on deployment is required for any real guarantee. Add uptime monitoring against the
health endpoint.

### 4. Inbound member resolution is non-deterministic — **high**

`Identify Member` matches on phone with `LIMIT 1`. Every production member and every trainer
in the current data shares a single phone number (the gym owner's). A reply is attributed to
whichever row the planner returns first, and that can change between runs.

**Always confirm which member an inbound test resolves to before trusting it.** Production
needs unique member phone numbers enforced, or the query disambiguated.

### 5. Per-case outreach cap is a lifetime cap — **medium**

`max_automated_outreach` counts messages for a case, not for a member over a period. A
member whose case is resolved and later re-flagged receives a fresh allowance of 3.

If a member churns and returns, they can be messaged 3 messages per episode indefinitely.
Consider counting per member over a rolling window instead.

### 6. No automated test suite — **medium**

Nothing in this repository runs in CI or locally without a live database. Every verification
described in these documents was performed by hand. This is the main obstacle to safe
iteration.

Minimum valuable additions: the ownership-drift query, the cooldown/cap behaviour, and the
four case-action permission checks.

### 7. `ai_analysis` is never written — **medium**

The AI classifies every reply but the row is never persisted, so there is no record of why
the system escalated or auto-replied. The table and constraints exist; the insert is missing.
Blocks any audit of AI decisions and any per-member AI cost measurement.

### 8. `risk_scores` is never written — **low**

Risk is computed in Code nodes and stored only on `retention_cases`. The table with four
sub-score columns, indexes and constraints exists and is empty. Either wire up the insert or
drop the table — an always-empty table misleads.

### 9. `error_logs` is empty despite observed failures — **low**

Workflow failures are visible in n8n execution history but do not reach `error_logs`. The
observability table is therefore not trustworthy.

### 10. Schedule interval values are missing — **low**

All four scheduled workflows store `{"interval":[{"field":"minutes"}]}` with no value. They
run every 5 minutes because that is the default, confirmed against execution timestamps —
but opening and saving any of them in the editor could bake in a different interval. Set the
value explicitly next time one is edited.

### 11. `retention_actions` CHECK lists `case_reopened` twice — **cosmetic**

Harmless duplicate in the constraint array literal. Worth cleaning up.

---

## 11. Production checklist

- [ ] Rotate every credential listed in § 9; delete the plaintext files
- [ ] Add webhook authentication to both nodes; send the header from the API
- [ ] `retryOnFail` on all 25 Postgres nodes
- [ ] Remove `NODE_TLS_REJECT_UNAUTHORIZED: "0"`
- [ ] Terminate TLS; restrict ports 5678 and 8080
- [ ] Always-on host with uptime monitoring
- [ ] Enforce unique member phone numbers
- [ ] Decide the outreach cap semantics (per case vs rolling window)
- [ ] Add a regression test suite
- [ ] Wire up `ai_analysis` and `risk_scores` inserts, or drop those tables
- [ ] Set schedule interval values explicitly
- [ ] Back up `n8n-postgres` (`postgres_data` and `n8n_data` volumes) and the app database
- [ ] Configure Brevo/Resend sender domain and verify deliverability
- [ ] Verify WhatsApp Business templates and the Evolution API session survive a restart