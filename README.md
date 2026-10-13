# HYNOVA Retain

**Member retention for gyms.** HYNOVA Retain scores every member for churn risk, opens a
retention case when one crosses a threshold, reaches the member on WhatsApp in their own
language, and holds the responsible trainer accountable until the case is closed.

It is not a messaging tool and not a CRM. It is the decision layer that sits on top of
membership and attendance data and answers one question: *which members is this gym about
to lose, and who is doing something about it?*

---

## Documentation

| Document | Contents |
|---|---|
| [docs/product-spec.md](docs/product-spec.md) | What the product does, who uses it, the rules it enforces |
| [docs/architecture.md](docs/architecture.md) | How the pieces fit together, request lifecycle, security model |
| [docs/database.md](docs/database.md) | Complete schema reference — 16 tables, constraints, triggers, indexes |
| [docs/workflows.md](docs/workflows.md) | All 6 n8n workflows — node graphs, schedules, webhook contracts |
| [docs/ai.md](docs/ai.md) | The AI layer — model, the 5 prompts verbatim, escalation logic, guardrails |
| [docs/deployment.md](docs/deployment.md) | Setup, configuration, ports, operations runbook, troubleshooting |

---

## Architecture at a glance

```
┌──────────────┐      HTTPS      ┌──────────────┐      SQL       ┌──────────────────┐
│   React SPA  │ ──────────────▶ │ Fastify API  │ ─────────────▶ │  PostgreSQL      │
│  (Vite, R19) │ ◀── JSON/JWT ── │   (port 3000)│ ◀───────────── │  (Supabase, RAG) │
└──────────────┘                └──────────────┘                └──────────────────┘
                                     │
                                     │  POST /webhook/gym-retention-case-action
                                     │  (server-side only — never from the browser)
                                     ▼
                              ┌──────────────┐      SQL       ┌──────────────────┐
                              │     n8n      │ ─────────────▶ │  PostgreSQL      │
                              │ (port 5678)  │ ◀───────────── │  (same DB)       │
                              └──────┬───────┘                └──────────────────┘
                                     │  POST /message/sendText/hynova-retain
                                     ▼
                              ┌──────────────┐
                              │ Evolution API │ ──▶ WhatsApp
                              │  (port 8080)  │
                              └──────────────┘
```

The application never talks to n8n from the browser. The Fastify API is the only caller of
the automation layer, which keeps the webhook URL and any future credential server-side and
gives one place to enforce authorisation.

### Components

| Component | Technology | Port | Role |
|---|---|---|---|
| `frontend/` | React 19, TypeScript, Vite 8, TanStack Query, Tailwind 4 | 5173 (dev) | Two role-separated UIs: gym staff and trainers |
| `backend/` | Fastify 5, TypeScript, `pg`, `bcrypt`, `@fastify/jwt` | 3000 | REST API, authentication, scoping, the n8n proxy |
| `n8n/` | n8n 2.x + Postgres 16 | 5678 | 6 automation workflows on a 5-minute cycle |
| Evolution API | `evoapicloud/evolution-api` | 8080 | WhatsApp Business gateway |
| Database | Supabase Postgres 15+, `pgvector` enabled | remote | 16 application tables, 3 triggers, 93 CHECK constraints |

---

## Repository layout

```
HYNOVA Retain - Gym Retention System/
├── frontend/                 React SPA
│   └── src/
│       ├── components/       layout/, ui/ primitives
│       ├── context/          auth, theme
│       ├── hooks/            data-fetching hooks
│       ├── lib/              api client, formatting, design tokens
│       ├── pages/            23 route components
│       ├── services/         typed endpoint wrappers
│       ├── App.tsx           route table (lazy-loaded)
│       └── index.css         Tailwind 4 + component layer
├── backend/                  Fastify API
│   └── src/
│       ├── db/
│       │   ├── migrations/   3 idempotent .sql files
│       │   ├── ensure-schema.ts  idempotent boot-time schema guard
│       │   └── postgres.ts   pooled connection
│       ├── hooks/            auth.ts, scope.ts
│       ├── routes/           14 files, 69 endpoints
│       ├── services/         email transport
│       ├── app.ts            Fastify instance + plugins
│       └── server.ts         entrypoint
├── n8n/
│   └── docker-compose.yml    n8n + postgres + evolution-api on `gym-net`
├── docs/                     this documentation set
├── scripts/
│   └── activate-n8n.ps1
└── Supabase/                 ⚠ contains live credentials — see deployment.md
```

---

## Quickstart

Full detail, including prerequisites and every environment variable, is in
[docs/deployment.md](docs/deployment.md).

```bash
# 1. Automation stack (n8n + Postgres + Evolution API)
cd n8n
docker compose up -d

# 2. API
cd ../backend
npm install
# populate .env  (see deployment.md)
npm run dev            # http://localhost:3000

# 3. Web app
cd ../frontend
npm install
npm run dev            # http://localhost:5173
```

Database schema is applied automatically at API boot by `ensureSchema()` — there is no
separate migration step for a fresh deployment.

---

## How the system actually works

Six n8n workflows run on a 5-minute cycle. Four are scheduled checkers, two are webhooks.

| Workflow | Trigger | What it does |
|---|---|---|
| **Retention Detection** | every 5 min | Scores every active member, opens a case when one crosses a threshold |
| **Retention Outreach** | every 5 min | Sends one AI-written WhatsApp message, respecting cooldown and cap |
| **Follow-up & Case Resolution** | every 5 min | Follows up on ageing cases; escalates to a trainer when needed |
| **Trainer Workflow** | every 5 min | Reminds trainers about cases they have not actioned |
| **Inbound Response Engine** | webhook | Handles member replies: classifies intent, auto-replies or escalates |
| **Case Resolution** | webhook | Trainer actions — contact, resolve, snooze, reassign |

The four scheduled workflows are **checkers, not senders**. Each run evaluates
eligibility and acts at most once per case. A member who has been contacted once is not
contacted again until the cooldown expires and the outreach cap allows it — regardless of
how many cycles elapse.

Full detail in [docs/workflows.md](docs/workflows.md).

### The retention loop

```
attendance + membership data
        │
        ▼
   risk score (0–100) ──▶ crosses threshold? ──no──▶ stop
        │ yes
        ▼
   retention case opened, assigned to the member's trainer
        │
        ▼
   cooldown + cap check ──▶ AI writes a WhatsApp message in the member's language
        │
        ▼
   member replies? ──yes──▶ AI classifies intent ──▶ auto-reply, or escalate to trainer
        │
        ▼
   trainer acts: contact / snooze / resolve / reassign
        │
        ▼
   case closed, full audit trail retained
```

---

## Security model

- **Passwords** are bcrypt-hashed. No plaintext is stored or logged.
- **Sessions** are JWTs. `users.active` is re-checked on **every request**, so
  deactivating an account takes effect immediately rather than at token expiry.
- **Identity is server-derived.** Action records take the actor from the verified token.
  The API rejects client-supplied `performed_by_id`; an audit entry cannot be forged.
- **Scoping is enforced in SQL.** A trainer may only reach cases whose member is assigned
  to them *and* belongs to the trainer's gym. Out-of-scope requests return `404`, not `403`,
  so the API does not confirm that an out-of-scope case exists.
- **Invariants live in the database.** 93 CHECK constraints mean an invalid status, risk
  level, action type, or role/trainer combination is rejected by Postgres regardless of
  which code path attempted it.
- **Automation credentials never reach the browser.** Only the Fastify API calls n8n.

See [docs/architecture.md](docs/architecture.md#security-model) for detail.

---

## Current status

Working and verified end-to-end: detection, outreach, cooldown enforcement, inbound reply
handling, trainer escalation, and all four trainer case actions through the UI.

Not yet production-ready. The known gaps are listed, with severity, in
[docs/deployment.md](docs/deployment.md#known-issues). The most important:

1. **Both webhooks are unauthenticated.** Anyone who can reach port 5678 can inject a
   member reply or a trainer action. Close this before exposing the system.
2. **No retry on Postgres nodes.** The database is remote (Supabase over WAN). 25 of 25
   Postgres nodes have `retryOnFail: false`, and transient failures drop outreach silently.
3. **No automated test suite.** Everything above was verified by hand against a live
   database.
4. **Plaintext credentials are committed to the repository.** See
   [docs/deployment.md](docs/deployment.md#credential-hygiene).