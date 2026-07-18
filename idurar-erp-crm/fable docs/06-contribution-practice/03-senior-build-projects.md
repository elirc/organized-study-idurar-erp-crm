# Senior Build Projects

6 projects, 2 days to 4 weeks, that a maintainer might actually accept. Each has a problem statement, product value, design checklist, architecture decisions, likely files, migration plan, test plan, security plan, performance plan, rollout/rollback, open questions, and stretch goals. These are portfolio-grade.

---

## Project 1: Payment integrity overhaul (1–2 weeks)

**Problem:** the payment flow is non-atomic and non-idempotent; `credit` can drift and retries double-count.

**Product value:** correct financial data — the single most important property of an invoicing app.

**Design checklist:** transaction boundaries; idempotency key scheme; reconciliation job; where status math lives (currently duplicated 3×).

**Architecture decisions:** transactions (needs replica set) vs derive-on-read; idempotency key source and storage; centralize status derivation into one function.

**Likely files:** `paymentController/{create,update,remove}.js`, `Invoice.js` (index for idempotency key), a new `reconcile` script, DB config for replSet.

**Migration plan:** deploy replSet-capable Mongo first; add idempotency-key unique index; ship transaction wrapping; run reconciliation once to fix existing drift.

**Test plan:** failure-injection integration tests; concurrent-payment test; idempotency-replay test; reconciliation test.

**Security plan:** idempotency keys must be per-user-scoped so one user can't replay another's.

**Performance plan:** transactions add latency; measure; derive-on-read is the fallback if replSet is unavailable.

**Rollout/rollback:** feature-flag the transaction path; rollback removes the flag (writes still work, just non-atomic).

**Open questions:** is a replica set available in the target deployment? If not, derive-on-read.

**Interview story potential:** the flagship story — "I made an invoicing system's money handling correct under failure and concurrency" (Stories 2, 7).

---

## Project 2: Test harness + CI foundation (1 week)

**Problem:** no tests, no CI — every change is manually verified.

**Product value:** the enabler for every other safe change; reduces regressions.

**Design checklist:** framework choice; DB isolation (memory server, replSet mode); module-alias mapping; which flows to cover first; CI pipeline shape.

**Architecture decisions:** Jest vs Vitest for backend; integration-first (given the Express coupling) vs unit-first.

**Likely files:** `backend/jest.config.js`, `backend/test/*`, `.github/workflows/ci.yml`, `package.json` scripts.

**Migration plan:** add deps → config → 5 integration tests (login, invoice create, payment invariant, overpay, cascade) → CI.

**Test plan:** the deliverable is tests; success = green CI on PRs.

**Security/perf plan:** n/a (tooling), but CI should run `npm audit`.

**Rollout/rollback:** additive; rollback removes the workflow.

**Open questions:** does the memory-server replSet mode work in the CI runner?

**Interview story potential:** "I introduced the first automated testing and CI to a legacy codebase" (Story 8).

---

## Project 3: Authorization & multi-tenancy foundation (2–4 weeks)

**Problem:** no per-record authorization; adding a second user creates an IDOR.

**Product value:** unlocks multi-user/multi-tenant — a major product capability.

**Design checklist:** tenant model (organization entity?); how scoping is enforced centrally; role model beyond `owner`; data migration for existing single-tenant data.

**Architecture decisions:** filter-injection middleware vs base-query helper; add `org` to every model vs `createdBy` scoping; how the JWT carries tenant context.

**Likely files:** new `Organization` model, `isValidAuthToken.js` (inject tenant), a shared query helper, all controllers, indexes on scope fields.

**Migration plan:** create a default org; backfill existing docs with it; add scope filters; tighten.

**Test plan:** cross-tenant read/write rejected; same-tenant works; migration correctness.

**Security plan:** this *is* the security project — the central risk is a forgotten filter, so enforce it in one place and test the negative case exhaustively.

**Performance plan:** scope fields need indexes; measure list latency post-scoping.

**Rollout/rollback:** high blast radius — feature-flag, canary, and keep the single-tenant path until proven.

**Open questions:** is multi-tenancy actually a product goal, or just multi-user within one org? Changes the model.

**Interview story potential:** "I designed the authorization foundation to take a single-tenant app multi-user" (system design + security).

---

## Project 4: Async job & queue infrastructure (2 weeks)

**Problem:** PDF generation and email run synchronously on the request thread; the counter bump is fire-and-forget.

**Product value:** responsiveness under load; reliable background work.

**Design checklist:** queue choice (BullMQ/Redis); which work moves to the queue (PDF, email); retry/backoff; result storage.

**Architecture decisions:** in-process worker threads vs external queue; where PDF results are cached (S3).

**Likely files:** `pdfController`, `downloadHandler`, `sendMail`, new worker + queue setup.

**Migration plan:** introduce the queue; move PDF generation behind it with a cache; move email; keep sync fallback during rollout.

**Test plan:** job enqueued/processed; retry on failure; cached result served.

**Security plan:** signed URLs for cached PDFs (ties to the download-auth fix).

**Performance plan:** measure event-loop lag before/after; the whole point is unblocking the loop.

**Rollout/rollback:** feature-flag the async path; rollback to sync.

**Open questions:** is Redis available in the deployment?

**Interview story potential:** "I moved blocking work off the request path with a job queue" (performance/reliability).

---

## Project 5: Observability stack (1 week)

**Problem:** near-zero observability; failures are often silent.

**Product value:** you find out about breakage before customers do.

**Design checklist:** structured logging (pino), request logging, metrics (event-loop lag, request latency, error rate), health check, error aggregation (Sentry).

**Architecture decisions:** what to log, correlation ids across the payment writes, what to alert on.

**Likely files:** `app.js` (middleware), `errorHandlers.js` (structured logging + stop leaking), new `/health`, config.

**Migration plan:** add logging → health → metrics → alerts, incrementally.

**Test plan:** health check reflects DB state; errors are logged with context and not leaked to clients.

**Security plan:** never log passwords or tokens; scrub request bodies.

**Performance plan:** logging overhead is minimal; sample high-volume logs if needed.

**Rollout/rollback:** additive; rollback removes middleware.

**Interview story potential:** "I instrumented a blind system, prioritizing the flows that failed silently" (ops).

---

## Project 6: TypeScript (or JSDoc + checkJs) adoption (2–4 weeks)

**Problem:** no static types; runtime `ReferenceError`s and magic strings that types would catch.

**Product value:** fewer regressions, better refactoring safety, editor support.

**Design checklist:** full TS migration vs incremental JSDoc + `// @ts-check`; shared types for the API envelope; how to handle the CJS/ESM split.

**Architecture decisions:** JSDoc-first is realistic here (no file renames, incremental); a shared schema (Zod) could generate both runtime validation and types.

**Likely files:** `tsconfig`/`jsconfig`, gradually annotated modules, a shared types package for the envelope.

**Migration plan:** enable `checkJs` on a few files → add `@typedef`s → expand. Don't big-bang.

**Test plan:** type-check passes in CI; the three known type-catchable bugs are caught.

**Security plan:** n/a.

**Performance plan:** build-time only.

**Rollout/rollback:** incremental by file; rollback per file.

**Open questions:** appetite for a build step on the backend (currently unbundled).

**Interview story potential:** "I introduced incremental type safety to a large JS codebase" (architecture).

---

## Choosing a project to actually build

If you want *one* portfolio piece: **Project 1 (payment integrity)** produces the strongest interview narrative and touches the most senior concepts (transactions, idempotency, reconciliation). But **Project 2 (tests + CI)** should come first in reality, because it makes Project 1 safe to ship. A great answer to "what would you do first" names Project 2 as the enabler and Project 1 as the payoff.
