# System Design From This Repo

Reverse-engineer IDURAR into a whiteboard exercise. The prompt an interviewer would give:

> "Design a simplified invoicing app: businesses log in, manage clients, create invoices with line items, record payments against invoices, and download PDF copies. Walk me through it."

You have an advantage: you've read a real implementation. Use it — but critique it, don't just recite it. At each step this doc shows **what IDURAR actually chose (with anchors)**, **a stronger/simpler alternative**, and **what junior / mid / senior answers sound like**.

Budget ~35 minutes: 5 requirements, 8 API + data model, 10 the hard parts (consistency, concurrency), 7 scaling, 5 wrap.

---

## Step 1 — Clarify requirements (5 min)

Always start here. Interviewers dock points for jumping to a schema.

Functional: auth; CRUD clients; CRUD invoices with embedded line items; record/edit/delete payments that update invoice balance; generate PDFs; a dashboard summary.

Non-functional to *ask about*: how many tenants (one business or many)? read/write ratio (invoicing is read-heavy)? consistency needs on money (strong)? PDF volume?

What IDURAR chose: **single-tenant, single `owner` role** (`backend/src/models/coreModels/Admin.js:30-34` — role enum is just `['owner']`). No organization/tenant entity. That's a scoping decision you should surface out loud: "IDURAR is single-tenant; I'll design for that first, then discuss multi-tenancy as an extension."

- **Junior:** starts drawing tables immediately.
- **Mid:** asks read/write ratio and whether money needs strong consistency.
- **Senior:** pins down the tenancy model first because it changes every subsequent decision (auth scoping, indexes, data isolation).

---

## Step 2 — API sketch (4 min)

IDURAR exposes a uniform per-entity REST surface, generated not hand-written (`backend/src/routes/appRoutes/appApi.js:8-26`):

```
POST   /api/:entity/create
GET    /api/:entity/read/:id
PATCH  /api/:entity/update/:id
DELETE /api/:entity/delete/:id
GET    /api/:entity/list?page&items&sortBy
GET    /api/:entity/search?q&fields
GET    /api/:entity/summary
POST   /api/invoice/mail        // entity-specific
```

Auth: everything except `/login`, `/forgetpassword`, `/resetpassword` sits behind a token check (`backend/src/app.js:39-41`).

Alternative to mention: REST-with-envelope (what IDURAR does, `{success,result,message}`) vs bare REST with HTTP-status semantics vs GraphQL/tRPC. IDURAR's envelope makes the client simple (`data.success` branch) but throws away HTTP semantics — a 202 can carry `success:false` (`backend/src/controllers/appControllers/paymentController/create.js:11-17`).

- **Junior:** lists endpoints.
- **Mid:** notes the envelope tradeoff and that mutations are POST/PATCH/DELETE.
- **Senior:** flags that a *generated* API means the resource contract is implicit (filename-driven) with no schema — and proposes an OpenAPI/tRPC contract for type safety.

---

## Step 3 — Data model (4 min)

Draw three core documents. IDURAR's real schema:

```
Admin(_id, email, name, role, enabled, removed)          // coreModels/Admin.js
AdminPassword(user→Admin, password, salt, loggedSessions[])// coreModels/AdminPassword.js
Client(_id, name, email, phone, createdBy→Admin, removed) // appModels/Client.js
Invoice(_id, number, year, client→Client, items[{itemName,qty,price,total}],
        subTotal, taxTotal, total, discount,
        credit, paymentStatus, payment[→Payment], status, createdBy)  // appModels/Invoice.js
Payment(_id, invoice→Invoice, client→Client, amount, date, createdBy)  // appModels/Payment.js
```

Three modeling decisions to call out with anchors:

1. **Line items are embedded, not a separate collection** (`Invoice.js:51-95`). Right call: items have no life outside their invoice, are always read together, and never queried independently. This is the textbook "embed when the child is owned and bounded" rule.
2. **Payments are referenced, not embedded** (`Invoice.js:126-131` holds an array of Payment ids). Right call: payments are created independently, edited, deleted, and listed on their own page.
3. **`credit` and `paymentStatus` are denormalized onto the invoice** (`Invoice.js:118-136`). A read-optimization: the invoice list shows paid-amount without aggregating payments. The cost is the consistency burden of Step 4.

Alternative: a normalized SQL design (Postgres) with `invoice_items` and `payments` tables and a computed balance via `SUM(payments.amount)`. Stronger consistency (transactions, foreign keys), weaker read performance without a materialized view. Naming this SQL-vs-document tradeoff is a strong-answer move.

- **Junior:** puts everything in one table or embeds payments too.
- **Mid:** embeds items, references payments, and can justify each.
- **Senior:** adds indexes (`{client:1}`, `{number:1, year:1}` unique) and explains the denormalization's consistency cost before being asked.

---

## Step 4 — The hard part: money consistency & concurrency (10 min)

This is where the interview is won. Two problems, both real in IDURAR.

### Problem A: cross-document consistency without transactions

Recording a payment mutates **two documents**: insert `Payment`, then update `Invoice.credit`/`paymentStatus`/`payment[]` (`paymentController/create.js:41` then `:66-77`). IDURAR does this as **three separate writes with no transaction**. If the process dies between them, the payment exists but the invoice balance is wrong — a broken invariant with no repair.

Your answer:
- State the invariant: `invoice.credit === SUM(non-removed payments.amount)` and `paymentStatus` is a pure function of `total - discount` vs `credit`.
- The fix: wrap both writes in a MongoDB transaction (`session = await mongoose.startSession(); session.withTransaction(...)`). Prerequisite: a replica set (transactions don't work on standalone Mongo) — say this; it shows depth.
- Cheaper alternative if transactions aren't available: make `credit` *derived on read* (aggregate payments) so there's a single source of truth and no invariant to break. Tradeoff: slower reads. Or a periodic reconciliation job that recomputes `credit` and flags drift.

### Problem B: concurrency / idempotency

Two concerns:
- **Invoice number races.** The number is seeded on the *client* from a stale `last_invoice_number` setting (`frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:40`) and the server bumps the counter fire-and-forget and un-awaited (`invoiceController/create.js:62-64`). Two users creating invoices at once can get the same number. Fix: allocate the number atomically server-side (`findOneAndUpdate({key},{ $inc:{value:1}},{new:true})` returns the new value) and use *that*, or accept duplicates and add a unique index `{number, year}` so the DB rejects collisions.
- **Payment retries.** A retried `POST /payment/create` creates a *second* payment and double-counts `credit` (no idempotency key). Fix: accept a client-generated idempotency key, unique-index it, and return the original result on replay.

- **Junior:** doesn't see the multi-write hazard.
- **Mid:** identifies "no transaction → inconsistent state on crash" and proposes a transaction.
- **Senior:** adds the idempotency-key design, the atomic counter, and knows transactions need a replica set — and weighs derive-on-read as the simpler alternative.

This step alone is the STAR story in [06-behavioral-star-stories.md](06-behavioral-star-stories.md) ("a subtle bug I found").

---

## Step 5 — Reads, PDFs, scaling (7 min)

- **Dashboard summary:** IDURAR uses one aggregation with `$facet` + `$lookup` (`clientController/summary.js:24-73`) — several stats in one round-trip, join done in the DB. Call out that the naive version (fetch clients, loop, query invoices each) is an **N+1**; the `$lookup` avoids it. Also note IDURAR's invoice summary silently ignores its own date range (`invoiceController/summary.js:27-42`, the date `$match` is commented out) — a correctness bug you'd fix.
- **PDF generation:** IDURAR renders Pug → `html-pdf` synchronously *on the request* and regenerates every time (`pdfController/index.js:25-79`). At scale this blocks the event loop and hammers disk. Fix: generate async in a worker/queue, cache the result in object storage (S3 — the repo already depends on `@aws-sdk/client-s3`), serve a signed URL.
- **Pagination:** offset-based `skip/limit` (`createCRUDController/paginatedList.js:37-39`). Fine for small data; degrades on deep pages (`skip` scans). Mention cursor/keyset pagination for large datasets.
- **Auth at scale:** the per-request `loggedSessions` lookup (`isValidAuthToken.js:32-47`) is a DB hit on every call. At scale, cache sessions in Redis or move to short-lived access tokens.

---

## Step 6 — Variation prompts

Interviewers pivot. Practice these out loud:

1. **"Now make it multi-tenant."** Add an `Organization`; scope every query by `orgId`; every model gets `org→Organization`; the auth middleware must inject and enforce the tenant filter (the current code has *no* per-record scoping — `paginatedList.js:32-41` returns all docs). This is the biggest change and the best test of whether you understood the single-tenant assumption from Step 1.
2. **"Add real-time: show payments live on the invoice."** WebSocket/SSE channel per invoice; publish on payment write; the denormalized `credit` becomes the thing you broadcast. Discuss delivery guarantees and reconnection.
3. **"10x the traffic."** Read replicas for lists/summaries; Redis for sessions and settings; move PDF + email to a queue; cursor pagination; add the missing indexes.
4. **"Support recurring invoices."** The schema already has a `recurring` enum (`Invoice.js:19-22`) — design the scheduler (cron/queue), idempotency per period, and how to avoid double-generating.
5. **"Make invoice edits auditable."** Append-only event log / change history table; discuss why mutating `update.js` in place loses history and how event sourcing would change the model.

---

## Scoring rubric for your own practice

| Signal | Weak | Strong |
| --- | --- | --- |
| Requirements | jumps to schema | pins tenancy + consistency first |
| Data model | one big table | embed items / reference payments, justified |
| Consistency | ignores multi-write | names invariant + transaction + prerequisite |
| Concurrency | silent | idempotency key + atomic counter |
| Scaling | "add servers" | specific: replicas, queue, cache, cursor pagination |
| Uses the repo | generic | cites what IDURAR chose *and* what to change |

If you can do Steps 1–4 fluently in 20 minutes with anchors, you are ready for a mid-level system-design round.
