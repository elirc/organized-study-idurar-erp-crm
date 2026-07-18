# Review Katas

8 fake PRs to review against this codebase. For each: read the author's intent and the diff, decide your findings (Blocking / Important / Optional), then write one kind, specific comment. Grade against the expected findings. All diffs are illustrative fake code; the "resembles" line points at the real code they'd touch.

Reviewing well means: does it work → is it correct → will it stay correct → does it fit the codebase → is it kind to the next maintainer.

---

## Kata 1: "Add email sending to invoices"

Author intent: wire up the `mail` endpoint to actually send.

Fake diff:
```js
// Illustrative fake code: not from this repo
const mail = async (req, res) => {
  const invoice = await Invoice.findById(req.body.id);
  await resend.emails.send({ to: invoice.client.email, subject: 'Invoice', html: renderInvoice(invoice) });
  res.json({ success: true });
};
```
Resembles: `invoiceController/sendMail.js` (currently a stub).

Expected findings —
- **Blocking:** no error handling — if `resend.send` throws, it rejects and (without `catchErrors` awareness) the client gets a raw 500 with the error object. Wrap or handle.
- **Blocking:** `invoice.client.email` assumes population and a present email — `Client.email` is optional (`Client.js:16`); guard it.
- **Important:** no auth/ownership check that the caller may mail this invoice (fine single-tenant, flag for multi-user).
- **Optional:** `renderInvoice` duplicates PDF templating; reuse `pdfController`.

Comment:
> Two things before this is safe: guard `client.email` (it's optional on the schema, `Client.js:16`) and handle a send failure so we don't 500 with the raw error. Also, can we reuse the Pug template in `pdfController` rather than a new `renderInvoice`?

---

## Kata 2: "Make invoice list faster by removing the count query"

Author intent: drop `countDocuments` to speed up lists.

Fake diff:
```js
// Illustrative fake code: not from this repo
const result = await Model.find({ removed:false }).skip(skip).limit(limit);
return res.json({ success:true, result, pagination:{ page, pages: 1, count: result.length } });
```
Resembles: `createCRUDController/paginatedList.js:32-56`.

Expected findings —
- **Blocking:** `count: result.length` is the *page* size, not the total — the frontend pagination (`redux/erp/actions.js:49-54`) uses `total` to compute page count; this breaks paging entirely.
- **Important:** if perf is the goal, use `estimatedDocumentCount()` (fast, uses metadata) instead of removing the count.
- **Optional:** measure first — is the count actually the bottleneck?

Comment:
> This breaks pagination — the client needs the *total* count to render page controls (`actions.js:49`), and `result.length` is just the current page. If `countDocuments` is slow at scale, `estimatedDocumentCount()` is the usual fix. Do we have a slow-query measurement showing this is the bottleneck?

---

## Kata 3: "Store JWT in a cookie instead of localStorage"

Author intent: improve security.

Fake diff:
```js
// Illustrative fake code: not from this repo
res.cookie('token', token, { httpOnly: true, secure: true });
res.json({ success: true, result: { ...user } }); // token no longer in body
```
Resembles: `authUser.js:40-53`, `request.js:16-25`.

Expected findings —
- **Blocking:** the frontend reads the token from localStorage to set the `Authorization` header (`request.js:20-24`) — moving it to an httpOnly cookie means the header is now empty and *every* API call 401s. This needs the backend to read the cookie (`isValidAuthToken` currently only reads the header, `:11-12`) and CSRF protection.
- **Important:** httpOnly cookies resist XSS but introduce CSRF risk — needs a CSRF token or `SameSite`.
- **Optional:** good direction, but it's a cross-cutting change, not a one-liner.

Comment:
> Good instinct on XSS, but this is bigger than it looks: `isValidAuthToken` reads the token from the `Authorization` header (`:11`), and the frontend sets that header from localStorage (`request.js:20`). If we move to an httpOnly cookie we need the server to read the cookie *and* add CSRF protection. Let's scope it as its own PR with those pieces.

---

## Kata 4: "Add a discount percentage to invoices"

Author intent: percentage discount on totals.

Fake diff:
```js
// Illustrative fake code: not from this repo
body.total = subTotal + taxTotal - (subTotal * req.body.discountPercent / 100);
```
Resembles: `invoiceController/create.js:37-43`.

Expected findings —
- **Blocking:** raw float math on money — use `calculate` (`helpers.js:47`).
- **Blocking:** `req.body.discountPercent` unvalidated — could be negative, >100, or a string; add a Joi rule (`schemaValidate.js`).
- **Important:** `paymentStatus` derivation (`create.js:45`) doesn't account for the new discount → status drift.
- **Optional:** persist `discountPercent` so it's reproducible, not just the computed total.

Comment:
> Let's route this through `calculate.multiply`/`calculate.sub` — raw float math will drift totals (`helpers.js:47`). And validate `discountPercent` (0–100) in `schemaValidate.js` before trusting it. One more: `paymentStatus` at line 45 needs to reflect the discount too.

---

## Kata 5: "Hard delete old soft-deleted invoices"

Author intent: reclaim space with a cleanup that hard-deletes anything `removed:true`.

Fake diff:
```js
// Illustrative fake code: not from this repo
await Invoice.deleteMany({ removed: true });
```
Resembles: soft-delete pattern in `createCRUDController/remove.js`.

Expected findings —
- **Blocking:** doesn't cascade — payments referencing these invoices (`Payment.invoice`) become dangling refs; autopopulate on those payments (`Payment.js:20-25`) will then populate `null`.
- **Important:** no age threshold — deletes *everything* soft-deleted, including recently deleted items a user might restore.
- **Optional:** should be a scheduled job with logging, not an ad-hoc endpoint.

Comment:
> This would orphan payments — `Payment.invoice` refs point at these ids and autopopulate would resolve to null. If we purge, we need to purge (or handle) the dependent payments too, and add an age threshold (e.g. removed > 90 days) so we're not deleting things users just removed. A logged scheduled job would be safer than an endpoint.

---

## Kata 6: "Fix the invoice number race with a random number"

Author intent: avoid duplicate invoice numbers.

Fake diff:
```js
// Illustrative fake code: not from this repo
body.number = Math.floor(Math.random() * 1000000);
```
Resembles: `InvoiceForm.jsx:40`, `invoiceController/create.js`.

Expected findings —
- **Blocking:** random numbers aren't sequential — invoices are legally/operationally expected to be sequential per year; this breaks the domain requirement and can *still* collide (birthday problem).
- **Important:** the real fix is atomic allocation (`findOneAndUpdate($inc,{new:true})`) or a unique `{number,year}` index with retry.
- **Optional:** —

Comment:
> Random breaks the sequential-numbering expectation (invoices are usually consecutive per year) and can still collide. The robust fix is to allocate the number atomically server-side with `$inc` returning the new value, and/or a unique index on `{number, year}` so the DB rejects any duplicate. Happy to pair on it.

---

## Kata 7: "Add current user to the response for the frontend"

Author intent: include the full admin in login response.

Fake diff:
```js
// Illustrative fake code: not from this repo
res.json({ success: true, result: { ...user.toObject(), password: databasePassword } });
```
Resembles: `authUser.js:40-53`.

Expected findings —
- **Blocking:** leaks the password document (hash + salt + all session tokens) to the client — a serious credential exposure. The current code deliberately returns only safe fields (`authUser.js:42-51`).
- **Important:** even `user.toObject()` may include fields you don't want (e.g. internal flags); return an explicit whitelist.
- **Optional:** —

Comment:
> This sends the password hash, salt, and every session token to the browser — that's a credential leak. The existing response returns an explicit safe subset (`authUser.js:42-51`); let's keep that shape and add only the specific field you need.

---

## Kata 8: "Speed up settings by caching in a module variable"

Author intent: cache settings in memory to avoid re-fetching.

Fake diff:
```js
// Illustrative fake code: not from this repo
let cachedSettings = null;
const loadSettings = async () => {
  if (cachedSettings) return cachedSettings;
  cachedSettings = await Setting.find({ removed:false });
  return cachedSettings;
};
```
Resembles: `middlewares/settings/loadSettings.js`.

Expected findings —
- **Blocking:** no invalidation — updating a setting (`updateBySettingKey`) won't refresh the cache, so the PDF/formatting uses stale values until restart. The `last_invoice_number` bump especially must not be stale.
- **Important:** module-level cache doesn't work across multiple server instances (each has its own copy).
- **Optional:** a short TTL or explicit bust-on-write would make it safe; `node-cache` (already a dep) supports TTL.

Comment:
> Caching settings is reasonable, but we need invalidation — `updateBySettingKey` and the `last_invoice_number` `$inc` would leave this stale until a restart, and stale currency/number settings cause visible bugs. A TTL (via `node-cache`, already a dep) or a bust-on-write hook would fix it. Note it also won't share across instances if we scale out.

---

## Self-grading rubric

| Level | Behavior |
| --- | --- |
| Basic | finds one issue, comment is vague ("this looks wrong") |
| Solid | separates blocking from optional, cites the file the change breaks |
| Strong | identifies the *systemic* issue (invalidation, cascade, whitelist), proposes the standard fix, and the comment is specific and kind |

Kind + specific is not optional politeness — a reviewer who is vague or harsh gets ignored or resented, and the bug ships anyway. See [07-career-and-collaboration/01-code-review-mindset.md](../07-career-and-collaboration/01-code-review-mindset.md).
