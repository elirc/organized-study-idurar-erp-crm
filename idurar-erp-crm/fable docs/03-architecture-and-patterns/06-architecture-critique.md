# Architecture Critique

What I'd say if a maintainer asked "you've read this for a week — what's your assessment?" This doubles as system-design interview material; cross-linked from [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

Each item is labeled **confirmed** (verified by reading) or **hypothesis** (plausible, needs runtime proof). Each recommendation has a migration path and a test strategy.

## Strongest design choices

1. **The generated CRUD layer.** Filename → routes → controller (`models/utils/index.js:22-47`, `appApi.js:8-31`) eliminates enormous boilerplate. Adding an entity is one file. For a fixed-shape CRUD app this is genuinely good engineering. **Confirmed.**
2. **Custom-controller escape hatch.** The factory falls back to generic handlers but lets invoices/payments override exactly what they need (`appControllers/index.js:16-33`, `invoiceController/index.js:12-18`). The right balance of DRY and flexibility. **Confirmed.**
3. **Server-authoritative money math.** Totals recomputed server-side (`invoiceController/create.js:30-43`), ownership stamped from the token (`:48`), payment max re-checked (`paymentController/create.js:30`). The security-critical instinct is correct. **Confirmed.**
4. **Dual-token sessions.** JWT + `loggedSessions` allow-list (`isValidAuthToken.js:45-53`) gets revocation that pure JWT can't. A thoughtful choice with a clear cost. **Confirmed.**
5. **Decimal-safe money via `currency.js`.** `helpers.js:47-60` avoids float drift where it's used. **Confirmed.**
6. **The single network chokepoint** (`request/request.js`) centralizes auth and error handling cleanly. **Confirmed.**

## Risks and tradeoffs, prioritized

### P0 — this copy is broken at boot
- **Frontend crashes on load** (`routes.jsx:73-116` references removed `Quote`/`PaymentMode`/`Taxes` with no imports). **Confirmed.** Fix first; it's tiny.
- **Setup script crashes** (`setup.js:54-67` requires missing models). **Confirmed.**

### P1 — correctness / security
- **Non-atomic payment writes** break the `credit` invariant on crash (`paymentController/create.js:41-77`). **Confirmed.** Highest-value real bug.
- **Unauthenticated PDF download** exposes business documents, ignores soft-delete (`app.js:42`, `downloadPdf.js:9-11`). **Confirmed.**
- **No authorization scoping** — fine for single-tenant, a critical IDOR the instant a second user exists (`paginatedList.js:32-41`). **Confirmed** (as a latent risk).
- **Invoice-number race** allows duplicates (`InvoiceForm.jsx:40` + un-awaited `$inc`). **Confirmed** by reading; duplicate occurrence is a **hypothesis** pending a concurrency test.
- **Invoice summary ignores its date range** (`invoiceController/summary.js:27-42`, date `$match` commented out) → "this month" sums all time. **Confirmed.**
- **Mass assignment** in generic update (`createCRUDController/update.js:9` spreads `req.body`). **Confirmed.**
- **ReDoS** via user-controlled regex in search (`search.js:17`). **Hypothesis** — shape is there, payload unproven.

### P2 — reliability / performance
- **PDF generated synchronously per request, no cache** (`pdfController/index.js:25-79`) — event-loop block + disk churn. **Confirmed.**
- **Global `autopopulate`** causes cascading over-fetch / hidden N+1 (`Invoice.js:178`, `Payment.js:56`). **Confirmed** as a pattern; perf impact is a **hypothesis** at current scale.
- **Unbounded `loggedSessions`** growth (`authUser.js:23`, no prune). **Confirmed.**
- **Missing indexes** on login email, invoice client/removed (`login.js:31`, `paginatedList.js:32`). **Confirmed** (absence), impact **hypothesis** scale-dependent.

### P3 — maintainability
- **No tests, no CI, no types.** Every change is manually verified. **Confirmed.**
- **Business logic coupled to `req`/`res`** — not unit-testable without a service layer. **Confirmed.**
- **Duplicated FE/BE money + formatting logic** (`helpers.js`/`utils/calculate.js`, both `useMoney`). **Confirmed.**
- **Status math duplicated** across payment create/update/remove — drift risk. **Confirmed.**

## "What I'd change owning this for 3 months"

Sequenced by blast radius and dependency:

| Phase | Change | Why now | Migration path | Test strategy |
| --- | --- | --- | --- | --- |
| Week 1 | Fix P0 crashers | app doesn't run | delete dead routes/nav, guard setup script | manual smoke: app loads, setup completes |
| Week 1–2 | Add a test harness (Jest + supertest + mongodb-memory-server) | nothing is safe to change without it | new dev deps + first integration test | the harness *is* the strategy — start with login + invoice create |
| Week 2–3 | Wrap payment writes in a transaction; add reconciliation job | the real data-integrity bug | requires replica set (docker-compose); `session.withTransaction` | reconciliation test asserting invariant after simulated failure |
| Week 3 | Atomic invoice-number allocation + unique `{number,year}` index | duplicates in prod | server-side `$inc` returning value; add index (online build) | concurrent-create integration test |
| Week 4 | Secure `/download` with signed URLs; add `removed` filter | data exposure | new authed endpoint issues signed URL; verify signature in route | test: expired/forged sig rejected; deleted invoice 404 |
| Week 5–6 | Extract a service layer for invoice/payment | unlock unit tests, decouple from Express | move logic to `invoiceService`, controller becomes adapter | unit tests on pure service functions |
| Week 6–8 | Introduce authorization scoping (even single-role, prep for multi-user) | close the IDOR before it exists | ownership filter in shared query helper | test cross-user read is rejected |
| Ongoing | Add indexes from slow-query logs; move PDF/email to a queue | perf under load | incremental | measure before/after |

**What I would NOT do:** a big-bang TypeScript migration or a rewrite of the generic reducer. In a test-less codebase, large refactors have huge blast radius and no safety net. Small, pattern-matching, individually-verifiable changes win.

## Interview framing

If asked "critique this architecture," lead with a strength (the generated CRUD layer is genuinely good), then the one risk that matters most (non-atomic payments), then your sequenced plan showing you'd stabilize (tests) before you improve (transactions) before you expand (multi-tenant). That arc — respect what works, fix what's dangerous, in the right order — is the senior signal.

## Drill

Pick any P1 item. Write the one-paragraph RFC a maintainer would need: problem, blast radius, proposed change, migration path, test plan, and what would make them reject it. Compare against the template in [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
