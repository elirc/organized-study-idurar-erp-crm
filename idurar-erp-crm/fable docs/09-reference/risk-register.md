# Risk Register

Every risk found while reading IDURAR, with evidence, anchors, impact, likelihood, suggested test, suggested fix, and confidence. **Confirmed** = verified by reading source. **Hypothesis** = plausible, needs runtime proof. Sorted by priority.

| # | Risk | Evidence / Anchor | Impact | Likelihood | Suggested test | Suggested fix | Confidence |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R1 | Frontend crashes on load | `routes.jsx:73-116` renders `<Quote/>`/`<PaymentMode/>`/`<Taxes/>` with no imports (`:5-26`) | app unusable | certain (every load) | load app, expect no `ReferenceError` | delete dead routes + nav links (Ticket 1) | **Confirmed** |
| R2 | Setup script crashes | `setup.js:54-67` requires missing `PaymentMode`/`Taxes` models | can't fully seed | certain on `npm run setup` | run setup, expect success | remove missing-model blocks (Ticket 2) | **Confirmed** |
| R3 | Non-atomic payment writes break `credit` invariant | `paymentController/create.js:41` then `:66-77`, no transaction | wrong financial data, silent | low per-request, inevitable over time | inject failure between writes, assert invariant | `withTransaction` (needs replica set) or derive-on-read | **Confirmed** (bug); drift occurrence **Hypothesis** |
| R4 | Unauthenticated PDF download | `app.js:42` (no auth), `downloadPdf.js:9-11` (no `removed` filter) | business docs exposed; deleted docs downloadable | high if URLs leak | request download without a token, expect it works today | signed expiring URLs + `removed:false` | **Confirmed** |
| R5 | No authorization scoping (IDOR latent) | `paginatedList.js:32-41`, `read.js:3-6` filter only `removed` | fine now; critical if multi-user | n/a single-tenant | seed 2 users, cross-read | central ownership filter middleware | **Confirmed** (as latent) |
| R6 | Duplicate invoice numbers | client seeds number (`InvoiceForm.jsx:40`); server bump un-awaited (`create.js:62`) | duplicate legal numbers | medium under concurrency | two concurrent creates, assert distinct | atomic `$inc` allocation + unique `{number,year}` index | **Confirmed** (mechanism); occurrence **Hypothesis** |
| R7 | Invoice summary ignores date range | `invoiceController/summary.js:27-42`, date `$match` commented out | wrong "this period" totals | certain when `type` used | seed cross-month, assert scoping | restore date `$match` (Ticket 9) | **Confirmed** |
| R8 | Mass assignment in generic update | `createCRUDController/update.js:9` spreads `req.body` | client can set unexpected fields | medium | send extra fields, assert ignored | whitelist updatable fields (Ticket 12) | **Confirmed** |
| R9 | JWT in localStorage (XSS exposure) | `redux/auth/actions.js:20`, `request.js:20-24` | token theft on any XSS | depends on XSS presence | n/a | short TTL / CSP / httpOnly cookie + CSRF | **Confirmed** (exposure) |
| R10 | Raw error echoed to client | `errorHandlers.js:22-28,71-77` | info disclosure (stack, internals) | every 500 | trigger 500, inspect body | generic prod message, log server-side (Ticket 6) | **Confirmed** |
| R11 | ReDoS via user regex | `search.js:17`, `paginatedList.js:15` | event-loop stall / DoS | unknown | time a crafted pattern | escape + length cap, or `$text` (Ticket 15) | **Hypothesis** |
| R12 | PDF generation blocks event loop, no cache | `pdfController/index.js:25-79` | latency spikes under load | medium under load | load-test concurrent downloads | async/queue + cache (Project 4) | **Confirmed** (mechanism); impact **Hypothesis** |
| R13 | Hidden N+1 via global autopopulate | `Invoice.js:178`, `Payment.js:56` cascade | slow lists at scale | scale-dependent | `mongoose.set('debug')`, count queries | explicit per-query populate | **Confirmed** (pattern); impact **Hypothesis** |
| R14 | `loggedSessions` grows unbounded | `authUser.js:23` `$push`, no prune | doc bloat over years | slow/certain | login many times, inspect array | prune expired / `tokenVersion` (Ticket 14) | **Confirmed** |
| R15 | Missing indexes on hot paths | `login.js:31`, `paginatedList.js:32` no declared indexes | slow queries at scale | scale-dependent | `explain()` → COLLSCAN | add indexes (Tickets 5, 13) | **Confirmed** (absence); impact **Hypothesis** |
| R16 | No rate limiting (login unthrottled) | `express-rate-limit` dep unused in `src` (grep) | brute-force / abuse | medium | hammer `/login`, observe no throttle | apply rate-limit middleware | **Confirmed** |
| R17 | `mail` endpoint is a silent stub | `invoiceController/sendMail.js:5-11` returns success | callers think mail sent | certain when used | call mail, observe no send | implement or clearly no-op (Ticket/Project 5) | **Confirmed** |
| R18 | Status-code/outcome mismatch | `paymentController/create.js:11-17` (202 for failure) | misleads status-based tooling | certain | assert status matches outcome | align to 400 (Ticket 3) | **Confirmed** |
| R19 | FE/BE money & format logic duplicated | `helpers.js` vs `utils/calculate.js`; both `useMoney` | drift between client/server | low/ongoing | compare outputs | shared package or ESM backend (Kata 4) | **Confirmed** |
| R20 | No tests / CI / types | absent in both packages | regressions ship undetected | ongoing | n/a | test harness + CI (Project 2) | **Confirmed** |

## How to use this register

- **Fix order:** R1, R2 (crashers) → R3, R4, R6, R7, R8 (correctness/security) → R9–R16 (hardening) → R19, R20 (foundations). This is the sequencing in [03-architecture/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md).
- **For interviews:** R3 (transactions), R4 (auth), R5 (IDOR), R6 (race), R13 (N+1) are your strongest technical stories.
- **Confidence discipline:** never present a **Hypothesis** as a confirmed bug. R11 (ReDoS) and the *impact* of R12/R13/R15 need runtime proof; the *mechanisms* are confirmed by reading.
