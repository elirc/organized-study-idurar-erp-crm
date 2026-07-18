# Trace Tables

Fill each table's blanks by reading the code, then check. Columns: **step, file/line, value shape, owner, transformation, risk**. These complement the key-flow tables in [01-codebase-cartography/05-key-flows.md](../01-codebase-cartography/05-key-flows.md) with finer-grained data-shape tracking.

---

## Trace 1: UI → API — creating an invoice (data shape at each hop)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `CreateItem.jsx:89-103` | form `fieldsValue` `{client, items:[{itemName,qty,price}], ...}` | React | compute `item.total` client-side | display only |
| 2 | `redux/erp/actions.js:69-78` | same + `entity:'invoice'` | thunk | wrap for request | — |
| 3 | `request/request.js:28-40` | JSON body | axios | POST `/invoice/create`, attach Bearer | — |
| 4 | `invoiceController/create.js:12-20` | validated `value` | controller | Joi check | 400 on fail |
| 5 | `create.js:30-43` | `body` with server `subTotal/taxTotal/total` | controller | **overwrite client totals** | — |
| 6 | `create.js:48-51` | `body` + `createdBy` | controller | stamp owner, `new Model().save()` | Mongoose validators |
| 7 | `create.js:52-59` | doc + `pdf` | controller | second write | — |
| 8 | `create.js:67-71` | `{success, result, message}` | controller | envelope | — |
| 9 | `redux/erp/actions.js:80-89` | `data.result` | thunk | dispatch SUCCESS + CURRENT_ITEM | — |
| 10 | `CreateItem.jsx:78-87` | — | React | on `isSuccess`, navigate to read page | — |

Verify: at which step does the client's `total` stop mattering? (Step 5 — server overwrites.)

---

## Trace 2: Persistence — soft delete cascade (invoice → payments)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `invoiceController/remove.js:7-17` | `{_id, removed:false}` | controller | `findOneAndUpdate` set `removed:true` | no transaction |
| 2 | `remove.js:19-25` | result or null | controller | 404 if not found | — |
| 3 | `remove.js:26-29` | `{invoice: id}` filter | controller | `Payment.updateMany` set `removed:true` | if this fails, invoice is deleted but payments aren't |
| 4 | `remove.js:30-34` | `{success, result}` | controller | envelope | — |

Verify: what's the inconsistent state if step 3 fails after step 1 succeeds? (Invoice removed, its payments still `removed:false` — orphaned live payments pointing at a dead invoice.)

---

## Trace 3: Auth — token from login to a protected request

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `authUser.js:15-21` | JWT string | backend | `jwt.sign({id})` | 24h/365d |
| 2 | `authUser.js:23-29` | — | backend | `$push` into `loggedSessions` | array grows |
| 3 | `redux/auth/actions.js:14-20` | `{current:{token,...}}` | frontend | write to localStorage | XSS-readable |
| 4 | `request/request.js:20-24` | header | frontend | `Authorization: Bearer <token>` | — |
| 5 | `isValidAuthToken.js:11-12` | token | backend | extract from header | — |
| 6 | `isValidAuthToken.js:22-30` | decoded `{id}` | backend | `jwt.verify` | 401 if bad |
| 7 | `isValidAuthToken.js:32-47` | user + loggedSessions | backend | verify membership | 401 if revoked |
| 8 | `isValidAuthToken.js:55-57` | `req.admin` set | backend | attach + `next()` | — |

Verify: which step makes logout actually work? (Step 7 — membership check; logout `$pull`s the token so step 7 fails.)

---

## Trace 4: Error path — Mongoose validation failure on create

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `create.js:51` | `new Model(body).save()` | controller | throws `ValidationError` (e.g. missing `currency`) | — |
| 2 | `errorHandlers.js:11` | error | wrapper | `.catch` receives it | — |
| 3 | `errorHandlers.js:12-19` | error.name === 'ValidationError' | wrapper | → 400 `{message:'Required fields...'}` | leaks `error` object (`:18`) |
| 4 | `request/errorHandler.js` | response | frontend | coerce to `{success:false}` | — |
| 5 | `redux/erp/actions.js:90-95` | — | thunk | dispatch REQUEST_FAILED | user sees notification |

Verify: does a Joi failure (`create.js:13-20`) take this path? (No — Joi returns 400 directly in the controller, never throws, so it bypasses `catchErrors`.)

---

## Trace 5: UI render — invoice list column formatting

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `redux/erp/actions.js:45-55` | `{items, pagination}` | thunk | shape API result for the table | — |
| 2 | `DataTable.jsx:47-49` | `listResult` | component | `useSelector` | re-render on change |
| 3 | `pages/Invoice/index.jsx:44-59` | `total` number | column render | `moneyFormatter({amount, currency})` | — |
| 4 | `settings/useMoney.jsx:31-35` | formatted string | hook | symbol before/after per settings | settings must be loaded |
| 5 | `DataTable.jsx:204-213` | rendered rows | Ant Table | commit to DOM | — |

Verify: what does the "paid" column actually read? (`record.credit`, `pages/Invoice/index.jsx:60-73` — labeled "paid" but bound to `credit`.)

---

## Trace 6: Payment overpay guard (server re-validation)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `RecordPayment.jsx:27-32` | `maxAmount = total-discount-credit` | frontend | compute for UX | client-side only |
| 2 | `RecordPayment.jsx:42-58` | body + invoice/client ids | frontend | submit | — |
| 3 | `paymentController/create.js:19-30` | invoice doc | backend | reload, recompute `maxAmount` | **server re-check** |
| 4 | `create.js:32-38` | — | backend | reject if `amount > maxAmount` (202) | — |
| 5 | `create.js:41-77` | payment + updated invoice | backend | the three writes | non-atomic |

Verify: why re-check on the server when the client already did (step 1)? (The client guard is bypassable; only step 3 is authoritative.)

---

## Meta-exercise

Pick Trace 1 and Trace 3. For each, circle the single step where a value crosses a **trust boundary** (client→server) and note what the server does to re-establish trust. (Trace 1: step 4-5, Joi + recompute. Trace 3: step 5-7, verify signature + session membership.) Recognizing trust boundaries in a trace is exactly the skill a code-review round tests.
