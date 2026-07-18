# Debugging & Code Review Rounds (Timed Simulations)

Four debugging scenarios and four review katas, framed as timed interview rounds with follow-ups and grading. These reuse real seams from this repo. Set a timer; narrate aloud — in a real round the interviewer grades your *process*, not just the answer.

The method to narrate every time: **reproduce → narrow → hypothesize → test cheaply → fix root cause → add a regression guard.** Say each step name out loud.

---

## Debug 1 (10 min): "Invoice balances are sometimes wrong after payments."

Symptom: some invoices show `credit` that doesn't match the sum of their payments.

Repo anchor: `paymentController/create.js:41, 66-77`; `remove.js:45-59`.

Narration path:
1. **Reproduce:** record a payment, kill the server mid-request (or imagine a thrown error after `Model.create`). Reload the invoice.
2. **Narrow:** is the payment row present? (Yes.) Is `invoice.credit` updated? (No.) → the gap is *between* the two writes.
3. **Hypothesize:** the two writes aren't atomic; a failure after write #1 leaves the invariant broken.
4. **Test cheaply:** grep for `startSession`/`withTransaction` — none. Confirmed: no transaction.
5. **Fix root cause:** wrap both writes in `session.withTransaction()` (needs replica set), or derive `credit` on read.
6. **Regression guard:** a reconciliation test asserting `credit === SUM(payments.amount)` after a simulated mid-flow failure.

Interviewer follow-ups: "What if it's not a crash but two concurrent payments?" (Lost update on `credit` unless `$inc` — it *is* `$inc`, so concurrent increments are safe; the crash window is the real bug.) "How do you find affected invoices in prod?" (The reconciliation query.)

Grade — Basic: blames the frontend. Solid: isolates the two-write gap. Strong: names the crash window, the transaction fix + its prerequisite, and a reconciliation guard.

---

## Debug 2 (10 min): "The app crashes on load with `ReferenceError: Quote is not defined`."

Symptom: white screen; console shows `Quote is not defined`.

Repo anchor: `frontend/src/router/routes.jsx:73-116` vs imports at `:5-26`.

Narration path:
1. **Reproduce:** load the app; read the stack trace → points at `routes.jsx`.
2. **Narrow:** the file references `<Quote/>`, `<QuoteCreate/>`, `<PaymentMode/>`, `<Taxes/>` but the `lazy(() => import(...))` declarations for them are absent (only the ones at `:5-26` exist).
3. **Hypothesize:** these verticals were removed from the backend and half-removed from the frontend; the route table still names them.
4. **Test cheaply:** search the file for `const Quote` — no match; `<Quote` — matches at `:73`.
5. **Fix root cause:** delete the dead route entries (and matching nav links in `NavigationContainer.jsx`). This is Ticket 1.
6. **Regression guard:** none automatable without a test setup, but a lint/build step would catch an undefined identifier — argue for adding one.

Follow-ups: "Why didn't the build catch this?" (No TypeScript; Vite/ESBuild doesn't error on undefined JSX identifiers at build, only at runtime.) "What would have prevented it?" (TS `noUndeclaredVariables`, or an eslint `no-undef`.)

Grade — Basic: comments out the whole routes file. Solid: removes exactly the dead entries. Strong: connects it to the missing type-checking and proposes the guard.

---

## Debug 3 (10 min): "Two invoices got the same number."

Symptom: a customer received two invoices both numbered 41.

Repo anchor: `InvoiceForm.jsx:40`, `invoiceController/create.js:62-64`, `increaseBySettingKey.js:11-31`.

Narration path:
1. **Reproduce:** open the create form in two tabs; both show `number = last+1`; submit both.
2. **Narrow:** where does the number come from? The client seeds it from a settings value read at page load (`InvoiceForm.jsx:40`), not from the server at submit.
3. **Hypothesize:** two clients read the same `last_invoice_number` before either increments → collision. The server's `$inc` is fire-and-forget and un-awaited (`create.js:62`), and swallows errors (`increaseBySettingKey.js:29-31`).
4. **Test cheaply:** confirm no unique index on `{number, year}` (schema has none) and no server-side allocation.
5. **Fix root cause:** allocate the number atomically server-side (`findOneAndUpdate($inc, {new:true})` returns the reserved value) and use it; or add a unique compound index and retry on duplicate-key error.
6. **Regression guard:** integration test firing two concurrent creates, asserting distinct numbers.

Follow-ups: "Which fix would you ship first?" (Unique index — smallest blast radius, DB enforces correctness immediately; then improve UX with server allocation.) "Why is the un-awaited `$inc` doubly bad?" (Response can return before it runs, and its errors vanish.)

Grade — Basic: "add a random suffix." Solid: sees the client-side read race. Strong: atomic allocation vs unique-index, picks by blast radius.

---

## Debug 4 (10 min): "Search occasionally makes the server unresponsive."

Symptom: certain search queries spike CPU and the whole API stalls.

Repo anchor: `createCRUDController/search.js:13-19`; single-threaded event loop.

Narration path:
1. **Reproduce:** search with a pathological string; watch CPU.
2. **Narrow:** the request builds `new RegExp(req.query.q, 'i')` from raw input (`search.js:17`). Node is single-threaded, so one expensive regex blocks *all* requests.
3. **Hypothesize:** ReDoS — catastrophic backtracking on an attacker-supplied pattern pins the event loop.
4. **Test cheaply:** feed a known evil pattern locally and time it. (Label *investigate* until demonstrated.)
5. **Fix root cause:** escape the input to a literal before building the regex, cap its length, or switch to a Mongo `$text` index for search.
6. **Regression guard:** a test asserting a crafted input returns within a time budget.

Follow-ups: "Why does one slow regex affect other users?" (Single event-loop thread; no request isolation.) "How do you protect the event loop generally?" (Offload CPU work to worker threads; input validation at the boundary.)

Grade — Basic: "add a try/catch." Solid: identifies event-loop blocking. Strong: names ReDoS, the escaping/`$text` fix, and event-loop offloading.

---

## Review Kata 1 (8 min): "Add a discount field to invoices"

Author intent: let users apply a percentage discount.

Fake diff summary:
```js
// Illustrative fake code: not from this repo
// invoiceController/create.js
const total = subTotal + taxTotal - req.body.discount;   // (1)
body.total = total;
const result = await new Model(body).save();
```
Files this resembles: `invoiceController/create.js:37-51`.

Your task: review it. Expected findings —
- **Blocking:** (1) uses raw `+`/`-` on money instead of `calculate` (`helpers.js:47-60`) → float drift; and `req.body.discount` is untrusted/unvalidated (could be a string, negative, or > subTotal → negative total).
- **Important:** `paymentStatus` recomputation (`create.js:45`) isn't updated for the new discount, so status math drifts.
- **Optional:** no Joi rule added in `schemaValidate.js` for `discount`.

Good review comment:
> Money math here should go through `calculate.sub`/`calculate.multiply` — raw `+`/`-` on floats will drift on totals (see `helpers.js:47`). Also let's validate `discount` (non-negative, ≤ subTotal) in `schemaValidate.js` before trusting it, and make sure `paymentStatus` at line 45 accounts for it.

---

## Review Kata 2 (8 min): "Speed up the client dashboard"

Author intent: show active-client count faster.

Fake diff summary:
```js
// Illustrative fake code: not from this repo
const clients = await Client.find({ removed: false });
let active = 0;
for (const c of clients) {
  const invoices = await Invoice.find({ client: c._id, removed: false });
  if (invoices.length) active++;
}
```
Files this resembles: the *correct* version at `clientController/summary.js:50-72`.

Your task: review it. Expected findings —
- **Blocking:** classic N+1 — one query per client (`await` in a loop). The existing code already solves this with a single `$lookup` aggregation (`summary.js:52-58`); this regresses it.
- **Important:** `await` inside a `for` serializes the queries, compounding the latency.
- **Optional:** doesn't reuse the `$facet` that computes the other stats in one pass.

Good review comment:
> This reintroduces an N+1 — for 500 clients it's 501 queries. The current aggregation does it in one round-trip via `$lookup` (`summary.js:52`). Let's keep the aggregation approach rather than looping; if you need a new stat, add a `$facet` branch.

---

## Review Kata 3 (8 min): "Make the delete endpoint faster"

Author intent: hard-delete instead of soft-delete to save space.

Fake diff summary:
```js
// Illustrative fake code: not from this repo
const result = await Model.findByIdAndDelete(req.params.id);
```
Files this resembles: `createCRUDController/remove.js:3-15` (which soft-deletes) and `invoiceController/remove.js:26-29` (cascade).

Your task: review it. Expected findings —
- **Blocking:** breaks the app's soft-delete invariant — everything else filters `removed:false` and expects records to persist; hard delete loses audit trail and orphans references (an invoice's `payment[]` ids now dangle).
- **Blocking:** skips the cascade — deleting an invoice must also soft-delete its payments (`invoiceController/remove.js:26-29`); hard delete leaves orphan payments.
- **Optional:** "saving space" isn't justified with data; premature.

Good review comment:
> This changes the delete semantics for the whole app. Everything reads `removed:false` and relies on records staying for audit/reference (payments point at invoices). Hard delete would orphan those refs and lose history. If storage is a real concern, let's add a separate purge job for old soft-deleted rows instead.

---

## Review Kata 4 (8 min): "Add current-user filter to lists"

Author intent: prep for multi-user by scoping lists to the owner.

Fake diff summary:
```js
// Illustrative fake code: not from this repo
// only in invoiceController/paginatedList.js
const results = await Model.find({ removed:false, createdBy: req.admin._id })...
```
Files this resembles: `invoiceController/paginatedList.js:23-33` and the generic `createCRUDController/paginatedList.js`.

Your task: review it. Expected findings —
- **Blocking:** scoping only the invoice list is a half-fix — `read/:id`, `update/:id`, `delete/:id`, and every *other* entity still return anyone's data. Authorization applied per-controller *will* be forgotten somewhere (that's the IDOR hole).
- **Important:** the scope belongs in shared middleware or a query helper so it can't be omitted.
- **Optional:** needs an index on `createdBy` to stay fast.

Good review comment:
> Good instinct, but scoping only `invoice/list` leaves `read`/`update`/`delete` and the other entities wide open — that's an IDOR waiting to happen. Let's put ownership scoping in one place (middleware or a base query) so every route inherits it, and add an index on `createdBy`.

---

## Self-grading for review katas

| Level | Behavior |
| --- | --- |
| Basic | finds one surface issue |
| Solid | separates blocking from optional, cites the pattern it breaks |
| Strong | proposes the systemic fix (centralize, don't patch one spot) and writes a kind, specific comment |
