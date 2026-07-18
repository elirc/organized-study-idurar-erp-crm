# Observability & Operations

The operations reality: this app has **almost no observability**. That's the lesson — "how would I know this broke?" mostly has the answer "I wouldn't, until a user complains." This file maps what exists, what's missing, and what to add per flow.

## What exists

- **Ad-hoc `console.log`.** Scattered (`CreateItem.jsx:90`, the welcome banners in `store.js`/`IdurarOs.jsx`, error branches). No structured logging, no levels, no correlation ids.
- **Error responses carry detail.** Every 500 returns the error message/object to the client (`errorHandlers.js:22-28`) — visible to the user, but not aggregated anywhere.
- **The friendly DB-connection error** (`server.js:21-26`) — a one-time boot log.
- **Redux DevTools** in dev (`store.js:20`) — client-state visibility only.

## What's missing (the whole list)

- No structured/leveled logging (no winston/pino).
- No request logging (no morgan) — you can't see traffic or latency.
- No metrics (no counters, no histograms, no event-loop-lag gauge).
- No tracing (no request correlation across the payment writes).
- No health check endpoint.
- No error aggregation (no Sentry) — errors vanish after the response.
- No CI, so no deploy/rollback signals from a pipeline.
- No uptime/alerting.

## "How would I know this broke?" per flow

| Flow | If it breaks today | What to add |
| --- | --- | --- |
| Login | user sees a 401/500 message; you learn from support | log auth failures with reason (not the password); rate-limit + alert on spikes |
| Payment invariant | **silent** `credit` drift | reconciliation job (`credit` vs `SUM(payments)`) run nightly, alert on mismatch |
| Invoice number | **silent** duplicate | unique index surfaces it as a write error you can log; alert on the error |
| PDF download | 500 to one user | metric on generation time + error rate; alert on p95 latency |
| Email (stub) | **silent** — returns success but sends nothing | remove the stub or log "mail not configured" |
| Settings load | whole app blocked on white screen | timeout + fallback; log settings-endpoint latency |
| DB connection | boot error log, then errors per request | health check + alert on connection state |

The pattern: the **silent** rows are the dangerous ones. A 500 the user sees is annoying but visible; silent data drift is the kind of bug that's discovered months later during an audit.

## What to add first (minimal, high-value)

1. **Request logging** (`morgan` or `pino-http`) — instant visibility into traffic, status codes, latency. One line in `app.js`.
2. **A health check** (`GET /health` returning DB connection state) — needed for any deploy/orchestration.
3. **Structured error logging** — log the error server-side with context *and* stop echoing it to the client (fixes an info-leak and gives you a record).
4. **The reconciliation job** — the one that catches the silent payment-drift bug. Even a manual script is better than nothing.

## Deploy / rollback (as present)

There's no deploy config, no Dockerfile referenced in the flows, no CI. Operationally this is "run `npm start` on a box with a Mongo URL." Rollback = redeploy the previous code. For a real deployment you'd want: a container, a health check the orchestrator polls, environment-specific config, and a migration step in the deploy (which requires adding migration tooling — see [03-architecture/02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md)).

## Blast radius awareness (an ops mindset)

Because there's no observability, the blast radius of a bad deploy is "unknown until users notice." That raises the bar on pre-merge care — the checklist in [05-security-checklist.md](05-security-checklist.md) and manual verification matter *more* here than in an observable system, because you can't lean on alerts to catch what you missed.

## Interview angle

"How would you know if feature X broke in production?" is a common ops question. Strong answer: name the signal (metric/log/alert) you'd want *per critical flow*, and flag the ones that would fail *silently* (the payment invariant here) as the highest-priority to instrument. That shows you think about failure detection, not just failure prevention.

## Drill

Pick the payment flow. Design the reconciliation job: what it queries, how often it runs, what it alerts on, and how it reports which invoices are affected. Then explain why this job is more important in *this* codebase than in one with database transactions. (Because without transactions, drift is expected to happen eventually, so detection is the safety net.)
