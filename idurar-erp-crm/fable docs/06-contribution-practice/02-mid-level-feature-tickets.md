# Mid-Level Feature Tickets

10 cross-layer tickets (schema/API/UI/tests). Each **requires design notes before implementation** and includes risk and rollback. These touch multiple layers — the mid-level skill is coordinating a change across the stack without breaking a contract.

Rule for every ticket: write the design note (data shape, API change, UI change, migration, test plan, rollback) *before* touching code. A maintainer will reject a cross-layer PR that changes the API contract without a note.

---

## Ticket 1: Make invoice numbering server-authoritative

Layers: backend controller, settings, frontend form.

Story: allocate the invoice number on the server atomically instead of on the client, eliminating duplicates.

Design notes required: how you allocate (`findOneAndUpdate` `$inc` returning the new value), how the frontend stops seeding `number` (`InvoiceForm.jsx:40`), and backward compatibility with existing invoices.

Acceptance: two concurrent creates get distinct numbers; the form no longer sends a client-chosen number (or the server ignores it).

Risk: changing where the number originates affects the create contract. Rollback: revert to client seeding (still works, just racy). Test: concurrent-create integration test.

Interview story potential: "eliminated a duplicate-numbering race with server-side atomic allocation" ([08/06](../08-interview-prep/06-behavioral-star-stories.md) Story 7).

---

## Ticket 2: Wrap payment writes in a transaction

Layers: backend controller, DB config.

Story: make payment create/update/delete atomic so the `credit` invariant can't break on partial failure.

Design notes: `mongoose.startSession()` + `withTransaction` around the payment + invoice writes (`paymentController/create.js:41,66-77`); the **replica-set prerequisite** and how dev runs it (docker-compose single-node replSet).

Acceptance: injected failure between writes leaves *neither* applied.

Risk: requires a replica set — standalone Mongo will error on transactions. Rollback: remove the session wrapping. Test: failure-injection integration test on a replSet memory server.

Interview story potential: Story 2 — "fixed a cross-document consistency bug." High value.

---

## Ticket 3: Add ownership scoping (prep for multi-user)

Layers: auth middleware, all controllers, indexes.

Story: scope reads/writes to `createdBy` (or a future tenant) in one shared place.

Design notes: a shared query helper or middleware that injects the ownership filter so no controller can forget it; index on `createdBy`; how single-role today migrates to multi-user later.

Acceptance: a request for another user's record is rejected (in a seeded two-user scenario).

Risk: applied per-controller it will be forgotten (that's the whole point of centralizing). Rollback: remove the filter. Test: cross-user read rejected.

Interview story potential: "closed an IDOR before multi-user shipped" (security, Story-worthy).

---

## Ticket 4: Async PDF generation with caching

Layers: backend handler, storage, frontend download.

Story: generate PDFs off the request thread and cache them so downloads don't re-render every hit.

Design notes: worker/queue vs a simple cache-on-first-generate; storage (local vs S3 — dep exists); cache invalidation on invoice update; the download route change.

Acceptance: repeated downloads don't regenerate; generation doesn't block other requests.

Risk: stale PDF if invalidation misses an update. Rollback: revert to synchronous generation. Test: second download served from cache; updated invoice busts cache.

Interview story potential: "moved blocking PDF work off the event loop" (performance).

---

## Ticket 5: Restore a working email flow

Layers: backend controller, config, frontend.

Story: replace the `sendMail` stub with real Resend email (with graceful "not configured" fallback).

Design notes: env-gated (`RESEND_API`); reuse the Pug template; error handling; what happens when the key is absent (clear no-op, not a fake success).

Acceptance: with a key, mail sends; without, a clear "email not configured" response (not a false success).

Risk: leaking client emails, unhandled send failures. Rollback: revert to stub. Test: mock the Resend client; assert send called / fallback taken.

Interview story potential: "replaced a silent stub with a real, safely-degrading integration."

---

## Ticket 6: Add pagination-safe cursor mode

Layers: backend list, frontend table.

Story: add optional keyset pagination for large collections alongside offset.

Design notes: `_id`-based cursor; how the frontend requests it; keeping offset as default for jump-to-page.

Acceptance: deep pages don't slow down; existing offset paging still works.

Risk: two pagination modes to maintain. Rollback: remove cursor path. Test: cursor returns non-overlapping pages.

Interview story potential: "added scalable pagination" (data/perf).

---

## Ticket 7: Extract invoice status/total math into a testable service

Layers: backend controllers.

Story: move the money + status logic out of the Express handlers into a pure `invoiceService`.

Design notes: function signatures taking plain data (no `req`/`res`); the controller becomes an adapter; unlocks unit tests.

Acceptance: controllers call the service; behavior unchanged; new unit tests cover the math.

Risk: subtle behavior change during extraction. Rollback: inline it back. Test: unit tests on the service (the whole point).

Interview story potential: "decoupled business logic from the framework to make it testable" (architecture/testing, Story-worthy).

---

## Ticket 8: Add a reconciliation job for the credit invariant

Layers: backend script/cron, observability.

Story: a job that recomputes `invoice.credit` from payments and reports mismatches.

Design notes: query design (aggregate payments per invoice, compare to `credit`); run cadence; how it alerts/reports; whether it auto-fixes or just flags.

Acceptance: seeded drift is detected and reported.

Risk: an auto-fix could mask the root cause. Rollback: disable the job. Test: seed a mismatch, assert it's flagged.

Interview story potential: "built the safety net that catches silent data drift" (reliability/ops).

---

## Ticket 9: First test harness + CI

Layers: tooling, CI.

Story: wire Jest + supertest + mongodb-memory-server and a GitHub Actions workflow.

Design notes: the module-alias mapping for Jest, the replSet memory server, which first tests to add ([05/02](../05-quality-engineering/02-writing-tests-here.md)), the CI steps.

Acceptance: `npm test` runs; CI runs on PRs (backend test + frontend lint/build).

Risk: flaky in-memory DB setup. Rollback: remove the workflow. Test: the harness *is* the deliverable.

Interview story potential: "introduced the first automated tests to a test-less codebase" (quality, high value).

---

## Ticket 10: Currency-consistency audit + fix

Layers: backend + frontend money math.

Story: find every raw-float money operation and route it through `calculate`.

Design notes: grep for `+`/`-` on amount fields (start with `paymentController/remove.js:39`); confirm each; the FE/BE duplication problem.

Acceptance: no raw float arithmetic on money remains; totals unchanged in tests.

Risk: touching many small spots. Rollback: per-spot revert. Test: money-math unit tests around edited spots.

Interview story potential: "audited and fixed floating-point money bugs across the stack."

---

## Meta-note

The single highest-value mid-level ticket is #9 (first tests + CI) — it unblocks safe delivery of every other ticket. In an interview, "the first thing I'd do is build a test harness so the rest is safe" is a strong, senior-leaning answer. Tickets #2, #7, and #8 together tell the complete payment-reliability story.
