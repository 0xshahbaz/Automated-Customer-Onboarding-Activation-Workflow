# Automated Customer Onboarding & Activation Workflow (n8n + PostgreSQL)

An event-driven activation tracking system that monitors new user signups through a SaaS product's onboarding funnel, automatically detects who's stalled, and routes intervention based on account value, an automated nudge email for standard users, a direct Slack alert to the Customer Success team for enterprise accounts.

Built to solve a problem every SaaS company has: new users sign up, then quietly drop off before reaching the product's "aha moment," and nobody notices until it's too late to help. This workflow watches every user's progress in real time and intervenes automatically, instead of relying on a human to catch it.

---

## What it does

1. A new user signs up (simulated via a `/signup` webhook) and is stored in a **PostgreSQL** (Supabase) database.
2. As the user progresses through onboarding, verifying their email, creating their first project, inviting a teammate, completing their first task, each milestone is logged as a timestamped event via a `/event` webhook, rather than overwriting a single status field.
3. A **daily scheduled job** runs a SQL query that reconstructs each user's current onboarding stage from their event history, then calculates how long they've been stuck there.
4. Users stalled for **3+ days** at any stage before full activation are flagged.
5. Stalled users are routed by account value:
   - **Enterprise accounts** → a Slack alert to the CS team, since these are high-value, high-risk-of-churn accounts that warrant personal outreach.
   - **Free/Pro accounts** → an automated, stage-specific nudge email.
6. A separate **funnel conversion report** query calculates what percentage of all signups reach each onboarding milestone, the kind of number a GTM or CS team would track weekly to see where users are dropping off.

---

## Why an event log, not a status table

Rather than storing one row per user with a single "current status" column, every milestone is logged as its own timestamped event in a separate `activation_events` table. This is a deliberate data modeling choice: an append-only event log preserves full history (when exactly did each step happen, not just whether it happened), which is what makes it possible to calculate time-based questions like "how long has this user been stuck" — something a flat status field can't answer on its own.

---

## Architecture

```
/signup Webhook
      │
      ▼
Insert into `users` table (Postgres)


/event Webhook
      │
      ▼
Select user by email (Postgres) — look up internal user ID
      │
      ▼
Insert into `activation_events` table (Postgres)


Schedule Trigger (daily)
      │
      ▼
Execute Query — reconstruct each user's milestone timeline
  (LEFT JOIN + conditional MAX aggregation across event types)
      │
      ▼
Code node — determine current stage + days stalled
      │
      ▼
IF: Is Stalled? (3+ days at current stage, not yet activated)
      │
      ├── No ──► (no action)
      │
      └── Yes
            │
            ▼
      IF: Is Enterprise account?
            │
      ┌─────┴─────┐
      ▼           ▼
  Enterprise   Free/Pro
      │           │
Slack: CS     Gmail: stage-specific
team alert    nudge email


Funnel Conversion Report (standalone query, run on demand)
  → signup → verified → project created → invited → activated,
    with conversion % at each stage
```

---

## Key design decisions

**One workflow, not three.** Signup intake, event logging, and scheduled stall detection all operate on the same two tables and serve one coherent system, so splitting them into separate workflows would add overhead without real benefit. Sticky notes on the canvas keep the three responsibilities visually distinct instead.

**Lookup-before-insert for events.** The `/event` webhook only receives an email, not an internal ID so a Select-then-Insert pattern resolves the external identifier into an internal one before logging the event.

**Stage detection via conditional aggregation.** `MAX(CASE WHEN event_type = '...' THEN occurred_at END)` per milestone turns a scattered event log into one row per user showing when each milestone happened — a standard, reusable pattern for reconstructing state from event data.

**Business-value-based routing.** A stalled enterprise account risks real revenue and gets a human via Slack; a stalled free/pro user gets an automated nudge. The routing reflects that difference rather than treating every stalled user the same.

---

## Tech stack

| Component | Tool |
|---|---|
| Workflow engine | [n8n](https://n8n.io) |
| Database | PostgreSQL, hosted via [Supabase](https://supabase.com) |
| Event intake | n8n Webhook nodes |
| Team notifications | Slack |
| User communication | Gmail |

PostgreSQL was chosen specifically over a no-code data store (like Airtable, used in earlier projects) to demonstrate real SQL: joins, conditional aggregation, and date-based calculations. skills that come up frequently in GTM/RevOps automation roles but aren't exercised by simple no-code database tools.

---

## Setup

1. Create a free Supabase project and run the schema below in the SQL Editor:

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  company TEXT,
  plan_tier TEXT DEFAULT 'free',
  signup_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE activation_events (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  event_type TEXT NOT NULL,
  occurred_at TIMESTAMP DEFAULT NOW()
);
```

2. In Supabase, use the **Connect** panel's **Transaction Pooler** connection details (not the direct connection, it resolves via IPv6 only and commonly fails to connect from n8n). Note the pooler host, port, and username format (`postgres.[project-ref]`).

3. In n8n, add a new **Postgres** credential using those pooler details.

4. Import the workflow JSON, and update:
   - The Postgres credential on every Postgres node
   - Slack `channelId` for the CS alert
   - Gmail sender/credential for the nudge email

5. Test end-to-end:
   - POST to `/signup` with a test user
   - POST to `/event` a few times with different `event_type` values for that user
   - Manually trigger the scheduled check (or wait for it) and confirm stalled users are correctly routed

---

## Known limitations / possible next steps

- **Nudge emails are not yet fully stage-specific.** The current email template is generic; a full version would use a Switch node on `stage` to send genuinely tailored copy per milestone.
- **No re-notification suppression.** A user stalled for 10 days would currently be flagged and nudged on both day 3 and every subsequent daily run — a production version would track "last nudged at" to avoid repeatedly messaging the same stalled user.
- **Funnel report is standalone, not scheduled or delivered anywhere.** Currently run manually; a natural next step would be scheduling it weekly and posting the result to Slack or a dashboard automatically.

---

## Screenshots
<img width="1878" height="764" alt="Screenshot 2026-07-07 183408" src="https://github.com/user-attachments/assets/89bb20ff-ddeb-4b9a-b6f8-5dd0686b9c48" />
