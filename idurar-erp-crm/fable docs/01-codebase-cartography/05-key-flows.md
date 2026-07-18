# Key Flows

Seven flows traced end to end. They rotate across the surfaces of the app — auth, generic CRUD, a money-mutating write, a read with server-side aggregation, a file download, a settings load, and the metaprogramming that wires it all together — so the rest of the curriculum can reference different flows instead of leaning on one favorite.

Read a flow with the cited files open. Predict each next step before you read it.

---

## Flow 1: Login and the session invariant

Why this flow matters: it is the app's only authentication boundary, and it stores sessions in a way most juniors have never seen — a JWT that is *also* tracked server-side. That combination is a classic interview topic (stateless vs stateful sessions).

Open these files first:
- `frontend/src/redux/auth/actions.js:5-31` — the login thunk.
- `frontend/src/auth/auth.service.js:7-27` — the axios call.
- `backend/src/controllers/middlewaresControllers/createAuthMiddleware/login.js` — validation and lookup.
- `backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js` — password check and token issue.
- `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js` — how the token is verified on every later request.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | `redux/auth/actions.js:11` | dispatch REQUEST_LOADING, call `authService.login` | `{email, password}` | — |
| 2 | UI | `auth/auth.service.js:9-12` | POST `/api/login?timestamp=…` | same | cache-buster only, not security |
| 3 | API | `routes/coreRoutes/coreAuth.js:8` | route → `adminAuth.login` | — | login is *not* behind `isValidAuthToken` (correct) |
| 4 | API | `login.js:13-29` | Joi validates email+password | validated `value` | invalid → 409 |
| 5 | API | `login.js:31-48` | find Admin by `{email, removed:false}`, check `enabled` | Admin doc | missing → 404; disabled → 409 |
| 6 | API | `authUser.js:5` | `bcrypt.compare(salt+password, hash)` | boolean | salt is prepended, not bcrypt's own salt — see note |
| 7 | API | `authUser.js:15-21` | `jwt.sign({id}, JWT_SECRET, {expiresIn})` | token string | 24h, or 365d if `remember` |
| 8 | API | `authUser.js:23-29` | `$push` token into `AdminPassword.loggedSessions` | array of tokens | array grows unbounded — see risk register |
| 9 | UI | `redux/auth/actions.js:20` | write `auth` object to `localStorage` | `{current, isLoggedIn,…}` | token in localStorage → XSS-readable |
| 10 | UI (later) | `request/request.js:16-25` | every request sets `Authorization: Bearer <token>` | header | — |
| 11 | API (later) | `isValidAuthToken.js:22-53` | verify JWT **and** check token is in `loggedSessions` | Admin doc on `req.admin` | dual check = server-side revocation |

Validation and authorization: input validation at `login.js:13-20` (Joi); authentication at `authUser.js:5-12`; per-request authorization at `isValidAuthToken.js:22-57`. Note the design choice at `isValidAuthToken.js:45-53`: a valid, unexpired JWT is still rejected if it is not in `loggedSessions`. That is what makes logout (`logout.js:11-18`, `$pull`) actually revoke a token — pure stateless JWT can't do that.

Tests that cover it: none found (no test files in repo).

What juniors usually miss: that the JWT alone is not trusted. They assume "valid signature = authenticated" and can't explain why the DB is consulted on every request.

What seniors notice: `loggedSessions` is an ever-growing array (`AdminPassword.js:30-33`) — every login on every device appends a token and nothing prunes expired ones. Over years this is unbounded document growth. Also, the salt is a random `shortid` stored in plaintext and *prepended to the password* before bcrypt (`authUser.js:5`, `setup.js:16-18`); bcrypt already salts internally, so this custom salt adds complexity without much benefit and is a talking point, not a vulnerability.

Interview angle: "Stateless JWT or server-side sessions?" — this repo does *both*, and you can explain the exact tradeoff (revocation vs a DB read per request) with `isValidAuthToken.js:45-53` as evidence. See [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q on auth.

Drill: disable your account by setting `enabled:false` in the DB while logged in. Predict, then verify, whether your existing token keeps working. (It does — `isValidAuthToken.js` never checks `enabled`. Only `login.js:43` does.) Write two sentences on the blast radius.

Self-grade — Basic: you can name the files. Solid: you can trace the token from sign to per-request check. Strong: you can explain why `loggedSessions` exists, what it costs, and one alternative (short-lived access token + refresh token, or a `tokenVersion` integer).

---

## Flow 2: Generic CRUD list (the convention engine)

Why this flow matters: 90% of this app's endpoints are not hand-written. Understanding the factory is the single highest-leverage thing you can learn here, and "how would you avoid writing the same CRUD 20 times?" is a real interview question.

Open these files first:
- `backend/src/models/utils/index.js:22-47` — builds `routesList` from model filenames.
- `backend/src/routes/appRoutes/appApi.js:8-31` — registers 9 routes per entity.
- `backend/src/controllers/appControllers/index.js:16-33` — picks custom controller or falls back to the factory.
- `backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js` — the `list` handler.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | boot | `models/utils/index.js:4` | glob `models/appModels/**/*.js` | `[Client, Invoice, Payment]` | filename *is* the contract |
| 2 | boot | `models/utils/index.js:28-47` | derive `entity`, `modelName`, `controllerName` | `routesList[]` | e.g. `Client.js` → `client`/`Client`/`clientController` |
| 3 | boot | `appControllers/index.js:7-10` | glob controller dirs | `[clientController,…]` | — |
| 4 | boot | `appControllers/index.js:29-33` | if no custom controller, `createCRUDController(modelName)` | controller object | — |
| 5 | boot | `appApi.js:28-31` | for each entity, call `routerApp` | Express routes | — |
| 6 | boot | `appApi.js:9-17` | wire `/entity/create`, `/read/:id`, `/list`, … | routes | all wrapped in `catchErrors` |
| 7 | request | `GET /api/client/list?page=1&items=10` | `paginatedList.js:1-8` | reads `page`,`items` | `page` is a string; `page*limit-limit` still works via coercion |
| 8 | request | `paginatedList.js:14-16` | build `$or` regex over `fields` | Mongo filter | regex from raw `q` — ReDoS/regex-injection surface |
| 9 | request | `paginatedList.js:19-29` | build `filterCondition`, reject object `equal` | filter | the object-guard added for NoSQL-injection safety |
| 10 | request | `paginatedList.js:32-50` | `find(...).skip().limit().sort()` + `countDocuments` in `Promise.all` | `[result, count]` | two round-trips, run in parallel |
| 11 | request | `paginatedList.js:57-71` | 200 with `{result, pagination}` or 203 if empty | JSON envelope | 203 for empty is unusual |

Validation and authorization: the whole `erpApiRouter` sits behind `isValidAuthToken` (`app.js:41`), but **there is no per-record ownership check** — any logged-in admin sees all documents. In a single-tenant app with one `owner` role (`Admin.js:30-34`) that is acceptable; in a multi-user app it would be a broken-access-control (IDORA) bug. This gap is a deliberate teaching seam.

Persistence and side effects: read-only. Note `paginatedList.js:40` calls `.populate()` with no args — a no-op that does nothing (contrast the invoice override at Flow 4).

Tests that cover it: none.

What juniors usually miss: that adding a file to `models/appModels/` silently creates a whole REST surface. They look for a routes file listing `/widget/create` and can't find it.

What seniors notice: the filename-to-route coupling is an *implicit contract* — renaming `Invoice.js` renames the API path and breaks the frontend. There is no compile-time check tying them together (no TS, no shared schema). That fragility is the price of the convention.

Interview angle: "How do you DRY up repetitive CRUD?" Answer with this factory (`createCRUDController/index.js:15-33`) and its cost (the implicit filename contract). See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: create `backend/src/models/appModels/Widget.js` with a `name` field, restart, and hit `GET /api/widget/list`. Predict the status code for an empty collection before you run it (203, `paginatedList.js:65`).

Self-grade — Basic: you know a factory exists. Solid: you can explain the model-file → route derivation. Strong: you can state the implicit contract, its failure mode (rename breaks the API), and how TypeScript or a shared schema registry would make it explicit.

---

## Flow 3: Custom controller override — create an invoice

Why this flow matters: it shows *why* a factory needs escape hatches. Invoices need money math and a side effect (bump the invoice-number counter) that generic CRUD can't provide.

Open these files first:
- `backend/src/controllers/appControllers/invoiceController/index.js` — how the override is assembled.
- `backend/src/controllers/appControllers/invoiceController/create.js` — the real create logic.
- `backend/src/controllers/appControllers/invoiceController/schemaValidate.js` — Joi contract.
- `backend/src/helpers.js:47-60` — the currency-safe `calculate` helpers.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | boot | `invoiceController/index.js:2` | start from `createCRUDController('Invoice')` | full CRUD object | inherits list/read/etc |
| 2 | boot | `invoiceController/index.js:12-18` | overwrite `create`,`update`,`delete`,`summary`,`list`,`read`,`mail` | patched object | only `search`,`filter`,`listAll` remain generic |
| 3 | request | `create.js:12-20` | Joi validate body | `value` | invalid → 400 with first message |
| 4 | request | `create.js:30-38` | per-item `total = qty*price`, accumulate `subTotal` via `currency.js` | numbers | float-safe math (`helpers.js:54`) |
| 5 | request | `create.js:37-47` | compute `taxTotal`, `total`, derive `paymentStatus` | numbers | `paymentStatus` from total-vs-discount |
| 6 | request | `create.js:48` | `body.createdBy = req.admin._id` | ObjectId | ownership stamped from the token, not the body |
| 7 | request | `create.js:51` | `new Model(body).save()` | Invoice doc | Mongoose `required` validators run here too |
| 8 | request | `create.js:52-59` | second write to set `pdf` filename `invoice-<id>.pdf` | updated doc | two writes; id not known until first save |
| 9 | request | `create.js:62-64` | `increaseBySettingKey('last_invoice_number')` — **not awaited** | fire-and-forget | ordering/atomicity gap — see below |
| 10 | request | `create.js:67-71` | 200 with the invoice | JSON | — |

Validation and authorization: two independent validation layers — Joi at `schemaValidate.js:2-24` (shape, required fields, item structure) and Mongoose `required`/`enum` at `Invoice.js` (e.g. `status` enum `:148-151`). Authorization is coarse (any admin). Ownership is *assigned server-side* at `create.js:48`, which is the right instinct — never trust a client-supplied `createdBy`.

Persistence and side effects: the counter bump at `create.js:62-64` is fired without `await`, so the HTTP response can return before (or even without) the counter incrementing; `increaseBySettingKey` swallows its own errors (`increaseBySettingKey.js:29-31`). Two invoices created concurrently can read the same `last_invoice_number` on the frontend and collide, because the number comes from the client form (`InvoiceForm.jsx:40`, `last_invoice_number + 1`), not from an atomic server-side allocation. This is a real race and an excellent interview story.

Tests that cover it: none.

What juniors usually miss: that `item.total` and the invoice `total` are computed **server-side and overwrite whatever the client sent** (`create.js:35-43`). The client math in `CreateItem.jsx:94-100` is only for display.

What seniors notice: invoice number generation is not idempotent or atomic. The counter, the number on the form, and the saved document can disagree under concurrency. A senior would allocate the number atomically (`findOneAndUpdate` with `$inc` returning the new value) and use it, or accept eventual duplicates and add a unique index — and would name that tradeoff.

Interview angle: "Where does business logic belong, and how do you keep money math correct in JS?" — point at `helpers.js:47-60` (never use raw `+` on floats) and `create.js:30-38`. See [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) (floating point) and 03 (number allocation).

Drill: annotate `create.js` marking every line that mutates `body`. Then answer: if Joi passes but the Mongoose save throws (e.g. missing `currency`), what does the client see? (500 via `catchErrors`, because the throw is caught by `errorHandlers.js:11-29`, not the Joi branch.)

Self-grade — Basic: you can point to where totals are computed. Solid: you can explain server-authoritative math and server-set ownership. Strong: you can describe the invoice-number race and a correct atomic allocation.

---

## Flow 4: Record a payment — the cross-document consistency invariant

Why this flow matters: this is the most important flow in the repo. A payment write must keep two documents consistent — `Payment` and its parent `Invoice` — with no database transaction. It is *the* system-design story here.

Open these files first:
- `frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:27-59` — client computes `maxAmount`, submits.
- `backend/src/controllers/appControllers/paymentController/create.js` — the two-document write.
- `backend/src/models/appModels/Invoice.js:118-136` — `credit`, `payment[]`, `paymentStatus`.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | `RecordPayment.jsx:27-32` | `maxAmount = total - discount - credit` | number | client-side guard only |
| 2 | UI | `RecordPayment.jsx:42-58` | attach `invoice`,`client` ids, dispatch `recordPayment` | payment body | — |
| 3 | API | `paymentController/create.js:11-17` | reject `amount === 0` | — | 202 (not 400) for a validation failure |
| 4 | API | `create.js:19-30` | load invoice, compute server-side `maxAmount` | numbers | **server re-checks** — good |
| 5 | API | `create.js:32-38` | reject `amount > maxAmount` | — | prevents overpayment |
| 6 | API | `create.js:39-41` | stamp `createdBy`, `Model.create(payment)` | Payment doc | **write #1** |
| 7 | API | `create.js:43-53` | second write to set `pdf` filename | updated payment | write #2 |
| 8 | API | `create.js:59-64` | recompute `paymentStatus` from `credit + amount` | string | paid / partially / unpaid |
| 9 | API | `create.js:66-77` | invoice `$push payment`, `$inc credit`, `$set paymentStatus` | updated invoice | **write #3** — the invariant |
| 10 | API | `create.js:79-83` | 200 with the payment | JSON | returns payment, not invoice |

Validation and authorization: server independently recomputes `maxAmount` (`create.js:30`) instead of trusting the client's — the correct pattern. Never rely on the UI's `RecordPayment.jsx:31` guard alone.

Persistence and side effects: **three writes, zero transaction.** If the process dies after `Model.create` (write #1) but before the invoice `$inc` (write #3), you have a payment that exists but is not reflected in the invoice's `credit` or `payment[]` — a broken invariant with no automatic repair. The `remove.js` and `update.js` paths mirror this with the same risk (`paymentController/remove.js:45-59`, `update.js:66-77`). Mongoose supports `session`/transactions on replica sets; this code uses none.

Tests that cover it: none — which is exactly why the invariant is fragile.

What juniors usually miss: that `credit` on the invoice is a **denormalized running total** — the sum of payment amounts, cached on the parent so lists don't have to aggregate. They treat it as a plain field and don't see that every payment mutation must maintain it.

What seniors notice: no atomicity across the three writes; no idempotency key (a retried request creates a duplicate payment and double-counts `credit`); status math is duplicated in three files (`create`/`update`/`remove`) and can drift. A senior would wrap the writes in a transaction, add an idempotency key, and centralize the status calculation.

Interview angle: this single flow answers "design an invoicing/payments system," "how do you keep denormalized data consistent," "what breaks without transactions," and "how do you make a POST idempotent." It is the backbone of [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: on paper, list the exact DB state after a crash between write #1 and write #3. Then write the 3-line fix using `mongoose.startSession()` and describe what it requires (a replica set). Then design an idempotency scheme.

Self-grade — Basic: you can find where `credit` changes. Solid: you can explain denormalization and the client-vs-server guard. Strong: you can enumerate the crash window, the retry double-count, and both fixes with their prerequisites.

---

## Flow 5: Dashboard read with server-side aggregation

Why this flow matters: it's the one place the app does real MongoDB aggregation (`$facet`, `$lookup`, `$group`) instead of a `find`. Aggregation and the N+1 tradeoff are common interview ground.

Open these files first:
- `backend/src/controllers/appControllers/clientController/summary.js:24-77` — client stats via `$facet`.
- `backend/src/controllers/appControllers/invoiceController/summary.js:33-80` — invoice totals + status breakdown.

Trace (client summary):

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | API | `clientController/summary.js:9-18` | validate `type` in week/month/year | string | invalid → 400 |
| 2 | API | `summary.js:24-73` | one `$facet` with 3 sub-pipelines | aggregation | single round-trip |
| 3 | API | `summary.js:50-72` | `activeClients` via `$lookup` into invoices | joined docs | join done in DB, not N queries |
| 4 | API | `summary.js:80-85` | compute percentages defensively (guard divide-by-zero) | numbers | `totalClients > 0 ?` guards |
| 5 | API | `summary.js:87-94` | 200 with `{new, active}` | JSON | — |

Validation and authorization: `type` whitelist at `summary.js:10-18` — a good example of allow-listing an enum instead of trusting input. Behind `isValidAuthToken`.

Persistence and side effects: read-only, single aggregation call. Contrast with what a naive implementation would do — fetch all clients, then loop and query invoices per client (the N+1 anti-pattern). The `$lookup` at `summary.js:52-58` is the fix baked in.

Tests that cover it: none.

What juniors usually miss: that `$facet` runs several independent aggregations over the same input in one pass, returning them as named arrays — so `totalClients`, `newClients`, `activeClients` cost one DB round-trip, not three.

What seniors notice: the invoice summary's date filter is commented out (`invoiceController/summary.js:36-41`), so "this month's total" actually sums *all* invoices regardless of `type`. The `type` param is validated but then ignored for the total — a latent correctness bug worth flagging (*investigate* → confirmed by reading: `startDate`/`endDate` are computed at `:27-29` but never used in the `$match`).

Interview angle: "What's an N+1 query and how do you avoid it?" — show the `$lookup`. "How do you compute several stats efficiently?" — show `$facet`. See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: rewrite the `activeClients` sub-pipeline as the naive N+1 version in pseudocode, then state how many DB round-trips each version makes for 500 clients (1 vs 501).

Self-grade — Basic: you recognize it's aggregation. Solid: you can explain `$facet` and `$lookup`. Strong: you spotted the unused date range and can describe the resulting wrong number plus the fix.

---

## Flow 6: PDF download (an unauthenticated boundary)

Why this flow matters: it's a public route that generates a file on demand, and it exposes a security boundary juniors routinely miss — the download endpoint has *no auth*.

Open these files first:
- `frontend/src/modules/ErpPanelModule/DataTable.jsx:93-95` — `window.open(DOWNLOAD_BASE_URL...)`.
- `backend/src/app.js:42` — `/download` mounted **without** `isValidAuthToken`.
- `backend/src/routes/coreRoutes/coreDownloadRouter.js:6-19` — parse id from filename.
- `backend/src/handlers/downloadHandler/downloadPdf.js` — render and stream.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | `DataTable.jsx:93-95` | open `…/download/invoice/invoice-<id>.pdf` in new tab | URL | no token attached — can't, it's a raw browser GET |
| 2 | API | `app.js:42` | route matched with **no auth middleware** | — | **anyone with the URL gets the PDF** |
| 3 | API | `coreDownloadRouter.js:8-9` | slice id out of the filename | id string | fragile string math |
| 4 | API | `downloadPdf.js:6-13` | look up model, `findOne({_id})` — no `removed:false` | doc | can render a soft-deleted invoice |
| 5 | API | `downloadPdf.js:23-38` | `generatePdf` renders Pug → html-pdf → temp file → `res.download` | file | regenerated every hit, no cache |
| 6 | API | `pdfController/index.js:25-27` | delete any existing file first, then recreate | file | disk write per request |

Validation and authorization: **none on this route.** The invoice `_id` is a MongoDB ObjectId (hard to guess, ~96 bits) so it's obscure, but obscurity is not authorization. Any party who ever saw a PDF URL can re-fetch it forever, including for soft-deleted invoices. This is the security kata in [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).

Persistence and side effects: regenerates the PDF from scratch on every request (`pdfController/index.js:25-79`), writing to `src/public/download/…`. No caching, no cleanup strategy beyond overwrite-on-next-request.

Tests that cover it: none.

What juniors usually miss: that `window.open` can't send the `Authorization` header, which is *why* the route is public — and that this is a design smell, not a necessity (a signed, expiring URL or a token query param would fix it).

What seniors notice: unauthenticated access to business documents, no `removed:false` filter (`downloadPdf.js:9-11`), and unbounded regeneration cost as a mild DoS vector. Also `html-pdf` (PhantomJS) is deprecated and unmaintained — a dependency-risk talking point.

Interview angle: "You have a file that only logged-in users should see, but the browser opens it in a new tab — how?" Signed URLs, short-lived tokens, or streaming through an authenticated XHR + blob. See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: describe two ways to make `/download` require auth without breaking `window.open`. Rank them by blast radius.

Self-grade — Basic: you found the download route. Solid: you can explain why it's public. Strong: you can name the missing `removed` filter, the DoS angle, and two auth fixes with tradeoffs.

---

## Flow 7: Settings load and money formatting

Why this flow matters: settings drive currency formatting, language, and the invoice-number seed. It shows a cross-cutting concern loaded once and read everywhere, and how the *same* settings object is consumed on both server (PDF) and client (UI).

Open these files first:
- `frontend/src/apps/ErpApp.jsx:36-38` — `settingsAction.list({entity:'setting'})` on mount.
- `frontend/src/settings/useMoney.jsx` — the formatter hook.
- `backend/src/middlewares/settings/loadSettings.js` and `increaseBySettingKey.js` — server-side reads/bumps.
- `backend/src/controllers/pdfController/index.js:34-68` — server reuses the same settings to format the PDF.

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | `ErpApp.jsx:36-38` | on layout mount, fetch all settings | settings list | blocks render until `isSuccess` (`ErpApp.jsx:42`) |
| 2 | UI | `redux/settings/*` | store settings, expose selectors | keyed object | — |
| 3 | UI | `settings/useMoney.jsx:8-13` | read `money_format_settings`, fall back to localStorage | format config | — |
| 4 | UI | `useMoney.jsx:31-35` | `moneyFormatter` places symbol before/after | string | driven entirely by settings |
| 5 | API (PDF) | `pdfController/index.js:34-57` | server loads settings, builds its own `moneyFormatter` | string | duplicated formatting logic FE/BE |
| 6 | API (create) | `invoiceController/create.js:62-64` | bump `last_invoice_number` setting | number | the counter seen in step 1 of Flow 3 |

Validation and authorization: settings live under `coreApiRouter` behind auth (`app.js:40`), except the values needed for PDFs are re-read server-side without a user context (`pdfController/index.js:34`).

Persistence and side effects: `increaseBySettingKey` (`:11-20`) is the only mutation, an atomic `$inc`. But as Flow 3 showed, the frontend seeds the invoice number from a *stale* copy of the setting, so the atomic bump doesn't prevent duplicate numbers.

Tests that cover it: none.

What juniors usually miss: that money formatting is implemented **twice** — once for the browser (`frontend/src/settings/useMoney.jsx`) and once for the PDF (`backend/src/settings/useMoney.js`) — and they can drift.

What seniors notice: duplicated formatting is a DRY violation across a process boundary that's hard to fix (browser and Node can't share the hook as-is); the render-blocking settings fetch (`ErpApp.jsx:42`, gated on `settingIsloaded`) means a slow settings call blocks the whole app shell.

Interview angle: "How do you handle app-wide configuration and i18n?" and "where do you draw the line on code duplication across frontend/backend?" See [08-interview-prep/02-frontend-framework-questions.md](../08-interview-prep/02-frontend-framework-questions.md).

Drill: find both `useMoney` implementations and list one formatting behavior that could differ between them if a maintainer edited only one.

Self-grade — Basic: you found the settings fetch. Solid: you can trace a currency symbol to a setting. Strong: you can articulate the FE/BE duplication and the render-blocking risk.

---

## Flow rotation map

The rest of the curriculum draws on these deliberately, so you see each surface more than once but never only one:

| Module | Primary flows used |
| --- | --- |
| 03-architecture/04-side-effects | Flows 3, 4 (transactions, idempotency) |
| 03-architecture/03-validation-auth | Flows 1, 2, 6 (auth boundary, IDORA gap) |
| 04-reading-gym | Flows 2, 3, 4, 5 (annotation + trace tables) |
| 05-quality/05-security | Flows 1, 6 (JWT storage, public download) |
| 05-quality/04-performance | Flows 2, 5, 7 (pagination, aggregation, render-block) |
| 08-interview/04-system-design | Flow 4 as the spine, Flows 2, 3, 5 as variations |
