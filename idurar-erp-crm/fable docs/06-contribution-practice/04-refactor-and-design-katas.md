# Refactor & Design Katas

Senior katas. Each is a design exercise, not a tutorial — do the thinking, then self-grade. No code is required for most; the deliverable is a design or an argument.

---

## Kata 1: Fix a boundary leak

Task: the money/status logic is coupled to `req`/`res` in `invoiceController/create.js`. Design the extraction of a pure `invoiceService.create(data, adminId)`.

Deliverable: the function signature, what it returns (result or throws a typed error), how the controller becomes a thin adapter, and the list of unit tests the extraction unlocks.

Self-grade — Strong: your service takes plain data (no HTTP objects), returns a domain result, and you can name ≥4 unit tests (totals, tax, status derivation, empty items) that become possible.

---

## Kata 2: Design the outbox pattern for the counter bump

Task: the invoice-number bump is fire-and-forget and can be lost (`invoiceController/create.js:62-64`). Design an outbox so the bump is guaranteed and ordered with the create.

Deliverable: where the outbox record is written (same transaction as the invoice), how it's processed, and how idempotency is preserved.

Self-grade — Strong: the outbox write shares the invoice's transaction, a processor drains it with at-least-once + idempotency, and you explain why fire-and-forget can't guarantee this.

---

## Kata 3: Split the "generic" DataTable

Task: `DataTable.jsx` claims to be generic but special-cases invoices (`:102-105`). Design the split so entity-specific behavior is injected, not hardcoded.

Deliverable: how invoice-specific actions (record payment) get injected via `config`/`extra`, and what the generic component no longer needs to know.

Self-grade — Strong: the generic table knows nothing about invoices; entity actions come entirely through props; you preserve current behavior.

---

## Kata 4: Remove the FE/BE money-logic duplication

Task: `calculate` and `useMoney` exist on both sides. Design deduplication across the CJS/ESM boundary.

Deliverable: extract a shared package that emits both module formats, or move the backend to ESM; name the blocker and the migration.

Self-grade — Strong: you address the CJS/ESM incompatibility head-on (not "just import it") and pick a realistic path with a rollback.

---

## Kata 5: Improve type safety without a full TS migration

Task: prevent the `keyState` magic-string bug and the undefined-component crash using the lightest tooling.

Deliverable: the JSDoc `@typedef` + `// @ts-check` approach vs full TS; which bugs each catches; the incremental adoption order.

Self-grade — Strong: you show a concrete `@typedef` for the `keyState` union and explain why `checkJs` catches the `routes.jsx` crash without renaming files.

---

## Kata 6: Design a migration for a required field

Task: make `Invoice.notes` required on a collection of a million existing invoices (many lacking it).

Deliverable: the deploy sequence (add optional → backfill → tighten), the backfill script shape, and how you avoid downtime.

Self-grade — Strong: three-phase deploy, backfill batched to avoid locking, and you note there's no migration framework so you'd add one first.

---

## Kata 7: Reduce the autopopulate N+1

Task: global `autopopulate` causes cascading joins on lists. Design the fix without breaking the pages that rely on populated data.

Deliverable: which refs lose global autopopulate, which queries add explicit `.populate()`, and how you verify no page breaks.

Self-grade — Strong: you replace global autopopulate with per-query populate only where needed, and you name how you'd measure the query-count improvement (`mongoose.set('debug')`).

---

## Kata 8: Write the RFC for securing downloads

Task: write the one-page RFC to make `/download` authenticated without breaking `window.open`.

Deliverable: problem, proposed design (signed expiring URLs issued from an authed endpoint), migration, test plan, rejection criteria. Use the template in [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).

Self-grade — Strong: your RFC picks the smallest-blast-radius option, includes verifying forged/expired signatures are rejected, and states what would make a reviewer say no.

---

## Kata 9: Review a flawed PR (design level)

Task: a colleague proposes "cache all settings in a module variable forever" to speed things up. Write the design-level review.

Deliverable: the invalidation problem, the multi-instance problem, and the safe alternative (TTL / bust-on-write via `node-cache`).

Self-grade — Strong: you identify staleness on `updateBySettingKey`/counter bump as the blocking issue and propose invalidation, not just "add a TTL."

---

## How to use these

Do one kata per week during the 8-week path. Write the deliverable as if it's a real design doc a maintainer will read — that *is* the senior skill these train. Katas 1, 2, 7 pair with the payment/perf projects; Katas 5, 8, 9 pair directly with interview rounds.
