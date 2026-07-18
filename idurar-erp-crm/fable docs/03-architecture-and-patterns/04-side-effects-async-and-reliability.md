# Side Effects, Async & Reliability

The reliability story of IDURAR, centered on its hardest problem: keeping two documents consistent without transactions.

## Map of side effects

| Side effect | Trigger | Location | Reliability property |
| --- | --- | --- | --- |
| Insert Payment + update Invoice | POST payment | `paymentController/create.js:41,66-77` | **non-atomic, 3 writes** |
| Soft-delete Invoice + its Payments | DELETE invoice | `invoiceController/remove.js:7-29` | non-atomic, 2 writes |
| Bump `last_invoice_number` | POST invoice | `invoiceController/create.js:62-64` | fire-and-forget, un-awaited, errors swallowed |
| Generate PDF to disk | GET download | `pdfController/index.js:25-79` | synchronous, regenerated per request |
| Second write to set `pdf` filename | POST invoice/payment | `invoiceController/create.js:52-59` | needs id from first write |
| Email (mail endpoints) | POST mail | `invoiceController/sendMail.js` | **stub** — returns "upgrade to premium" |
| File upload to disk/S3 | profile/setting update | `middlewares/uploadMiddleware/*` | multer disk storage |

## The core reliability problem: non-atomic payment

Recording a payment mutates two documents in three writes with no transaction:

```
1. Model.create(payment)                 // paymentController/create.js:41
2. Model.findOneAndUpdate(payment, {pdf}) // :43-53  (cosmetic)
3. Invoice.findOneAndUpdate(              // :66-77  THE INVARIANT
     $push payment, $inc credit, $set paymentStatus)
```

**Failure window:** if the process dies (or write #3 throws) after write #1, a Payment exists but `invoice.credit`, `invoice.payment[]`, and `invoice.paymentStatus` are stale. The invariant `credit === SUM(non-removed payments.amount)` is broken with no automatic repair. The same shape exists in `update.js` and `remove.js`.

**Why it usually works anyway:** the writes are milliseconds apart and failures are rare — so the bug is invisible until a crash, a deploy mid-request, or a DB blip. That's what makes it a *good* interview story: it's subtle, real, and has a clean fix.

## The reliability concepts and how they apply

### Atomicity
All-or-nothing. The payment flow needs writes #1 and #3 to both happen or neither. Fix: `mongoose.startSession()` + `session.withTransaction(() => { ... })`, passing `{session}` to each write. **Prerequisite: a replica set** — MongoDB transactions don't run on a standalone `mongod`. Say this in an interview; it shows you've actually done it.

Cheaper alternative that removes the invariant entirely: **derive `credit` on read** (aggregate payments) so there's a single source of truth and nothing to keep in sync. Tradeoff: slower reads (the very thing denormalization was avoiding).

### Idempotency
A retried request should have the same effect as one request. IDURAR has **no idempotency key**, so a client retry (timeout, double-click) creates a *second* payment and double-counts `credit`. Fix: accept a client-generated idempotency key, unique-index it, and on replay return the original result instead of writing again. This is a classic payments-system requirement and a strong senior answer.

### Retries and error visibility
The counter bump (`increaseBySettingKey.js:29-31`) *swallows its own errors* and is fire-and-forget (`create.js:62`, no `await`). So if it fails, nothing knows — the invoice number drifts silently and no log records it. Fix: at minimum log the failure; better, make the number allocation part of the atomic create.

### Ordering / races
The invoice number is read on the client from a stale setting (`InvoiceForm.jsx:40`) and bumped separately server-side. Two concurrent creates collide. Fix: atomic server-side allocation (`findOneAndUpdate($inc,{new:true})` returns the reserved number) or a unique `{number,year}` index that rejects duplicates.

### Backpressure / resource cost
PDF generation is synchronous on the request thread and regenerates every hit with no cache (`pdfController/index.js:25-79`). Under load this blocks the event loop and hammers disk — a mild DoS vector. Fix: generate async in a worker/queue, cache in S3 (already a dependency), serve a signed URL.

## Where side effects sit in risky places

- The counter bump happens *after* the response is effectively decided and isn't awaited — a side effect outside the request's success guarantee.
- PDF generation on a *public, unauthenticated* route (`app.js:42`) means anyone can trigger unbounded server work.
- The `mail` endpoint is a stub (`sendMail.js:5-11`) — a side effect that silently does nothing, which is its own reliability trap (callers think mail was sent).

## Failure-visibility exercise: "how would I know this broke?"

| Flow | Current signal if it breaks | Better signal |
| --- | --- | --- |
| Payment invariant | none — silent drift | reconciliation job + alert on mismatch |
| Invoice number | none — swallowed | log the swallowed error; unique index surfaces dup |
| PDF generation | 500 to the one user | metrics on generation time + error rate |
| Email | none — stub returns success | remove the stub or make it clearly a no-op |

## Drill

Write the transactional version of `paymentController/create.js` using `mongoose.startSession()`, and list what infrastructure it requires to run. Then design an idempotency-key scheme for the payment endpoint: where the key comes from, how you store it, and what you return on a replay. Self-grade — Strong: your transaction wraps writes #1 and #3, you name the replica-set prerequisite, and your idempotency design returns the *original* result on retry without a second write.

**Interview angle:** this file is the backbone of [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md) Step 4 and Debug 1/Debug 3 in [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).
