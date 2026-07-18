# Fake-Code Contrasts

10 novice-vs-better pairs. Every snippet is **illustrative fake code, not from this repo** — but each is tied to a real IDURAR pattern so you can see the instinct and the correction in context. Read the bad version, predict the problem, then read the better one.

---

## 1. Coupling UI shape to DB shape

Real pattern: `pages/Invoice/index.jsx` renders `dataIndex:['client','name']` from the populated doc.

```js
// Illustrative fake code: not from this repo — NOVICE
// The component assumes the raw Mongo document shape:
<td>{invoice.client._id.toString()}</td>
<td>{invoice.paymentStatus === 'partially' ? 'Partial' : invoice.paymentStatus}</td>
// If the API changes client to a string id, this crashes on .name
```

```js
// Illustrative fake code: not from this repo — BETTER
// Map the API doc to a view model at the boundary:
const row = toInvoiceRow(invoice); // { clientName, statusLabel }
<td>{row.clientName}</td>
<td>{row.statusLabel}</td>
```

Lesson: don't let the DB document shape leak into JSX. A view model decouples the UI from persistence changes. In this repo the coupling is mild (Ant column `dataIndex`) but real — renaming a field breaks columns silently.

---

## 2. Missing permission filter

Real pattern: `paginatedList.js:32-41` filters only `removed:false`.

```js
// Illustrative fake code: not from this repo — NOVICE
const invoices = await Invoice.find({ removed: false });
// Every user sees every invoice — fine for single-tenant, an IDOR the moment there are two users
```

```js
// Illustrative fake code: not from this repo — BETTER
const invoices = await Invoice.find({ removed: false, createdBy: req.admin._id });
// Or, better, inject this filter in shared middleware so no query can forget it
```

Lesson: authorization scoping belongs in one shared place, not sprinkled per-query where it's easy to omit. See [03-architecture-and-patterns/03-validation-auth-and-permissions.md](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).

---

## 3. N+1 query

Real pattern: `clientController/summary.js:50-72` avoids this with `$lookup`.

```js
// Illustrative fake code: not from this repo — NOVICE
const clients = await Client.find({ removed: false });
for (const c of clients) {
  c.invoiceCount = await Invoice.countDocuments({ client: c._id }); // 1 query PER client
}
```

```js
// Illustrative fake code: not from this repo — BETTER
const rows = await Client.aggregate([
  { $lookup: { from: 'invoices', localField: '_id', foreignField: 'client', as: 'invoices' } },
  { $addFields: { invoiceCount: { $size: '$invoices' } } },
]); // ONE round-trip
```

Lesson: `await` in a loop over query results is the N+1 smell. Join in the DB. 500 clients = 501 queries vs 1.

---

## 4. Stale cache key / derived state duplication

Real pattern: `ItemRow.jsx:9-52` shadows form state in `useState`.

```js
// Illustrative fake code: not from this repo — NOVICE
const [total, setTotal] = useState(0);
useEffect(() => { setTotal(price * quantity); }, [price, quantity]);
// total is derived data stored in state — two sources of truth that can drift
```

```js
// Illustrative fake code: not from this repo — BETTER
const total = calculate.multiply(price, quantity); // derive during render, no state
```

Lesson: don't store what you can compute. Derived state in `useState` drifts from its source and needs an effect to sync — a class of bug you delete by computing inline.

---

## 5. Side effect in an unreliable place

Real pattern: `invoiceController/create.js:62-64` bumps the counter fire-and-forget.

```js
// Illustrative fake code: not from this repo — NOVICE
const invoice = await Invoice.create(body);
bumpCounter(); // not awaited, errors swallowed — did it run? nobody knows
res.json({ success: true, result: invoice });
```

```js
// Illustrative fake code: not from this repo — BETTER
const session = await mongoose.startSession();
await session.withTransaction(async () => {
  const number = await allocateNumber(session); // atomic, part of the unit
  await Invoice.create([{ ...body, number }], { session });
});
```

Lesson: a side effect that must happen should be part of the transactional unit, not a fire-and-forget after the response is decided.

---

## 6. Swallowed error

Real pattern: `increaseBySettingKey.js:29-31` catches and returns null silently.

```js
// Illustrative fake code: not from this repo — NOVICE
try {
  await doImportantThing();
} catch { return null; } // failure vanishes with no trace
```

```js
// Illustrative fake code: not from this repo — BETTER
try {
  await doImportantThing();
} catch (err) {
  logger.error('doImportantThing failed', { err, context });
  throw err; // or return a typed failure the caller can act on
}
```

Lesson: swallowing an error means you'll never know it happened. At minimum log it; usually, propagate it.

---

## 7. Raw float money math

Real pattern: `paymentController/remove.js:39` uses raw `-`; the rest uses `calculate`.

```js
// Illustrative fake code: not from this repo — NOVICE
const remaining = invoice.total - invoice.discount - invoice.credit; // float drift
```

```js
// Illustrative fake code: not from this repo — BETTER
const remaining = calculate.sub(calculate.sub(invoice.total, invoice.discount), invoice.credit);
```

Lesson: once you touch money with raw `+`/`-`, drift creeps back in even if you used a decimal library elsewhere. Be consistent.

---

## 8. Trusting client-supplied identity

Real pattern: `invoiceController/create.js:48` sets `createdBy` from the token (correct); `createCRUDController/update.js:9` spreads `req.body` (risky).

```js
// Illustrative fake code: not from this repo — NOVICE
await Invoice.findByIdAndUpdate(id, req.body); // client can set createdBy, removed, credit...
```

```js
// Illustrative fake code: not from this repo — BETTER
const { itemName, quantity, price, notes } = req.body; // whitelist
await Invoice.findByIdAndUpdate(id, { itemName, quantity, price, notes });
```

Lesson: never spread an untrusted body into a write (mass assignment). Whitelist the fields a client may change.

---

## 9. Unanchored regex from user input

Real pattern: `search.js:17` builds `new RegExp(req.query.q)`.

```js
// Illustrative fake code: not from this repo — NOVICE
const rx = new RegExp(req.query.q, 'i'); // ReDoS + full collection scan
await Model.find({ name: { $regex: rx } });
```

```js
// Illustrative fake code: not from this repo — BETTER
const safe = escapeRegex(req.query.q).slice(0, 100); // escape + cap length
await Model.find({ name: { $regex: safe, $options: 'i' } });
// or use a $text index for real search
```

Lesson: untrusted regex is a denial-of-service vector on a single-threaded runtime. Escape it, cap it, or use full-text search.

---

## 10. HTTP status divorced from outcome

Real pattern: `paymentController/create.js:11-17` returns 202 for a validation failure.

```js
// Illustrative fake code: not from this repo — NOVICE
if (amount === 0) return res.status(202).json({ success: false, message: '...' });
// 202 Accepted for a rejected request — misleads every status-based tool
```

```js
// Illustrative fake code: not from this repo — BETTER
if (amount === 0) return res.status(400).json({ success: false, message: '...' });
// 400 Bad Request — status matches the outcome
```

Lesson: when you also have a `success` envelope, keep HTTP status aligned with it. Monitoring, retries, and CDNs read the status, not your body.

---

## Drill

Pick three contrasts. For each, find the *real* IDURAR line referenced and decide: does the repo do the novice thing, the better thing, or a mix? (E.g. #8 — it does the *better* thing for invoice create but the *novice* thing in generic update.) Being able to spot which side real code is on, per case, is the reviewer skill this trains.
