# Data Model & Persistence

## The entities and their relationships

```
Admin 1───1 AdminPassword          (credentials split out)
Admin 1───* Client                 (createdBy, assigned)
Admin 1───* Invoice                (createdBy)
Client 1───* Invoice               (invoice.client, autopopulated)
Invoice 1───* Payment              (payment.invoice + invoice.payment[])
Client 1───* Payment               (payment.client)
Setting                            (standalone key-value rows)
```

Anchors: `Admin.js`, `AdminPassword.js:11` (1:1 via unique `user` ref), `Client.js:21-22`, `Invoice.js:9,31-36,126-131`, `Payment.js:9-25`.

## Key modeling decisions

### Embedded line items (embed the owned child)
`Invoice.items` is an **embedded array** (`Invoice.js:51-95`), not a collection. Correct because items: (a) have no identity outside the invoice, (b) are always read with it, (c) are never queried across invoices. This is the "embed when owned and bounded" rule. Cost: you can't efficiently query "all invoices containing product X" — acceptable here since there's no product catalog.

### Referenced payments (reference the independent child)
`Invoice.payment` is an **array of refs** (`Invoice.js:126-131`); payments are their own collection. Correct because payments are created, edited, deleted, and listed independently. The parent keeps a ref list so it can show its payments without a reverse query.

### Denormalized balance (`credit`, `paymentStatus`)
`Invoice.credit` is a cached running total of payments; `paymentStatus` is a cached derived enum (`Invoice.js:118-136`). A read-optimization: the invoice list shows paid-amount and status without aggregating payments per row. **This is the central persistence decision and the source of the app's hardest bug** — the cache must be maintained on every payment mutation, and it's done non-transactionally (see [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md)).

### Credentials split from user
`AdminPassword` is separate from `Admin` (`AdminPassword.js`) so password hashes and session tokens never accidentally serialize with the user object (`authUser.js:40-53` returns the Admin without the password doc). Good separation.

## Transaction boundaries and consistency

**There are no transactions anywhere.** Multi-document writes happen as sequential independent operations:
- Payment create: insert Payment, then update Invoice (`paymentController/create.js:41,66-77`).
- Invoice delete: soft-delete Invoice, then soft-delete its Payments (`invoiceController/remove.js:7-29`).

Consistency expectation is "best effort, usually fine, broken on crash." The invariant `credit === SUM(non-removed payments.amount)` can silently break. To fix, Mongoose supports `session`/`withTransaction` — but **only on a replica set**, which the default single-node dev setup isn't. That prerequisite is the crux of the interview answer.

The one place consistency *is* concurrency-safe: `credit` updates use `$inc` (`paymentController/create.js:70`), an atomic operator, so two concurrent payments don't lose an update. The gap is the crash window between the two documents, not concurrent increments.

## Indexes

Almost none are declared. Present:
- `AdminPassword.user` is `unique` (`AdminPassword.js:11`) — enforces 1:1.
- `_id` on every collection (automatic).

Missing but needed on hot paths:
| Query | Field(s) | Anchor |
| --- | --- | --- |
| login | `Admin.email` | `login.js:31` |
| list/filter | `Invoice.removed`, `Invoice.client` | `paginatedList.js:32` |
| default sort | `{removed:1, enabled:-1}` compound | `paginatedList.js:6` |
| duplicate-number guard | `{number:1, year:1}` **unique** | `Invoice.js:10-17` (would also fix the race) |

Indexes cost write throughput and storage, so add them from real slow-query evidence — but the login email lookup and the invoice-number unique index are safe bets.

## Seeds and migrations

- **Seed:** `setup.js` creates the admin (`admin@admin.com`/`admin123`) and default settings. In this copy it **crashes** because it requires `PaymentMode`/`Taxes` models that were removed (`setup.js:54-67`) — the admin+settings are created first, so login may still work.
- **Migrations:** none. No `migrate-mongo` or equivalent. Schema changes are ad hoc. `upgrade.js`/`reset.js` exist but are setup helpers, not a migration framework.

## How to safely change the schema

1. **Additive optional field:** safe. MongoDB is schemaless at storage; old docs read the default. Add to the Mongoose schema with a `default`, done.
2. **New required field:** unsafe without a backfill. Deploy order: add as optional → backfill existing docs → tighten to `required` in a later deploy.
3. **Rename a field:** unsafe. Old docs have the old key. Add the new field, dual-write, backfill, then drop the old — or accept a migration script. Remember renaming a *model file* also renames the API route (the filename convention).
4. **Change an enum:** removing a value orphans existing docs holding it. Migrate those docs first.

Because there's no migration tooling and no tests, every schema change here is manual and risky — proposing a migration framework is a legitimate senior project (see [06-contribution-practice/03-senior-build-projects.md](../06-contribution-practice/03-senior-build-projects.md)).

## Drill

Write the Mongoose index declarations you'd add to `Invoice.js` and `Admin.js`, and for each state the query it accelerates and its write-side cost. Then write the safe deploy sequence to make `Invoice.notes` a required field on a collection with a million existing invoices.
