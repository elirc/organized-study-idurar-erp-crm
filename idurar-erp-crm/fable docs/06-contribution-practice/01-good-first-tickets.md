# Good First Tickets

16 junior tickets spread across the repo. Each is small, follows an existing pattern, and is verifiable by hand. Format is compressed after Ticket 1 (which shows the full template) — the fields are the same throughout.

---

## Ticket 1: Remove dead Quote/PaymentMode/Taxes routes

Difficulty: Easy — Estimated time: 30 min — Skills: reading routes, safe deletion.

Story: As a user, I want the app to load without crashing, so that I can use it at all.

Why good: fixes a confirmed P0 crash (`routes.jsx:73-116` references `<Quote/>`, `<PaymentMode/>`, `<Taxes/>` with no imports → `ReferenceError`). Tiny blast radius — pure deletion of already-broken code.

Acceptance criteria:
- [ ] App loads without `ReferenceError`.
- [ ] `/quote`, `/payment/mode`, `/taxes` no longer appear in the nav (`NavigationContainer.jsx`).
- [ ] No remaining reference to the removed components.

Read first: `frontend/src/router/routes.jsx:5-26` (imports) and `:73-116` (dead usages); `frontend/src/apps/Navigation/NavigationContainer.jsx` (nav items).

Files likely touched: `routes.jsx` (remove entries), `NavigationContainer.jsx` (remove links).

Plan: 1) delete the route objects for quote/paymentMode/taxes; 2) delete the matching nav `items`; 3) load the app, confirm no error and the remaining nav works.

Illustrative fake-code shape (labeled):
```jsx
// Illustrative fake code: not from this repo — remove blocks like:
{ path: '/quote', element: <Quote /> }, // <-- delete
```

What could go wrong: leaving a nav link whose route you deleted → 404 on click. Check both files.

Suggested checks: manual — load app, click every remaining nav item.

Review questions: did you remove *all* references, or just some? Any other file importing these pages?

Interview story potential: "I stabilized a codebase that wouldn't boot by removing half-deleted feature routes" (STAR story 1/8, [08/06](../08-interview-prep/06-behavioral-star-stories.md)).

---

## Ticket 2: Fix the crashing setup script

Difficulty: Easy — 30 min — Skills: Node, reading requires.

Story: As a developer, I want `npm run setup` to complete, so I can seed a fresh install.

Why good: fixes confirmed crash — `setup.js:54-67` requires `PaymentMode`/`Taxes` models that don't exist in this copy. Small, self-contained.

Read first: `backend/src/setup/setup.js:54-67`; `backend/src/models/appModels/` (only Client/Invoice/Payment).

Plan: remove (or guard) the `PaymentMode`/`Taxes` require+`insertMany` blocks; run `npm run setup`; confirm it prints success and creates the admin + settings.

What could go wrong: removing too much (the admin/settings seed must stay). What would make a maintainer reject it: deleting seed data other than the missing-model blocks.

Interview story potential: "made a broken onboarding script work" (mistake/ownership angle).

---

## Ticket 3: Align payment validation status codes to 400

Difficulty: Easy — 20 min — Skills: HTTP semantics.

Story: As an API consumer, I want validation failures to return 400, not 202, so status-based tooling is correct.

Why good: `paymentController/create.js:11-17,32-38` returns 202 for rejections; 400 is correct. Follows the pattern already used in `invoiceController/create.js:15`.

Read first: `paymentController/create.js:11-38`; contrast `invoiceController/create.js:13-20`.

What could go wrong: the frontend may branch on status somewhere — grep `202` in `frontend/src` first (it branches on `success`, so likely safe, but verify). Reject reason: changing status without checking client handling.

Interview story potential: "aligned HTTP semantics with the response contract" (API design).

---

## Ticket 4: Add a Joi rule for invoice `discount`

Difficulty: Easy — 30 min — Skills: Joi validation.

Story: As a maintainer, I want `discount` validated so it can't be negative or non-numeric.

Why good: closes a validation gap; follows the existing `schemaValidate.js` pattern.

Read first: `invoiceController/schemaValidate.js:2-24`; `Invoice.js:122-125`.

Plan: add `discount: Joi.number().min(0).optional()` to the schema. Verify a negative discount now returns 400.

Reject reason: making it `required` (breaks existing clients that omit it).

Interview story potential: "hardened input validation" (security/validation).

---

## Ticket 5: Add missing `Admin.email` index

Difficulty: Easy — 20 min — Skills: Mongoose indexes.

Story: As the login path, I want an index on `Admin.email` so lookups don't scan.

Why good: login queries `Admin.findOne({email})` (`login.js:31`) with no index. Additive, safe.

Read first: `models/coreModels/Admin.js:14-19`; `login.js:31`.

Plan: add `adminSchema.index({ email: 1 })`. Confirm with `.explain()` that login uses `IXSCAN`.

Reject reason: making it `unique` without confirming no duplicate emails exist (the schema allows removed duplicates).

Interview story potential: "added an index driven by a hot query path" (performance).

---

## Ticket 6: Stop echoing raw errors in production

Difficulty: Easy — 30 min — Skills: error handling, security.

Story: As a security-conscious maintainer, I want production 500s to hide internals.

Why good: `errorHandlers.js:71-77` echoes the `error` object. Small change, real info-leak fix.

Read first: `handlers/errorHandlers.js:51-77`.

Plan: in `productionErrors`, log the error server-side and return a generic message; keep detail in `developmentErrors`.

Reject reason: removing the message entirely (breaks legitimate client error display).

Interview story potential: "closed an info-disclosure gap" (security).

---

## Ticket 7: Add `removed:false` to the download lookup

Difficulty: Easy — 20 min — Skills: soft-delete awareness.

Story: As a maintainer, I want deleted invoices to be non-downloadable.

Why good: `downloadPdf.js:9-11` looks up by `_id` only, so soft-deleted invoices still render. One-line consistency fix.

Read first: `handlers/downloadHandler/downloadPdf.js:9-13`.

Plan: add `removed: false` to the `findOne`. Verify a soft-deleted invoice now 404s on download.

Reject reason: none likely — but note this doesn't fix the *auth* gap (that's a bigger ticket).

Interview story potential: part of the download-security story.

---

## Ticket 8: Remove unused `AbortController` dead code in DataTable

Difficulty: Easy — 20 min — Skills: reading effects.

Story: As a maintainer, I want misleading dead code removed.

Why good: `DataTable.jsx:162-168` and `ErpPanelModule/index.jsx:21-27` create an `AbortController` that's never wired to anything. Clarity fix.

Read first: `DataTable.jsx:162-168`.

Plan: either wire the abort to the dispatched request or remove it. For a first ticket, remove it and note the effect deps issue for a follow-up.

Reject reason: removing the effect itself (it does the initial load).

Interview story potential: minor — cleanup.

---

## Ticket 9: Fix the invoice summary date range

Difficulty: Medium — 45 min — Skills: aggregation, debugging.

Story: As a user, I want the dashboard "this month" total to actually scope to this month.

Why good: fixes a confirmed correctness bug — `invoiceController/summary.js:37-40` has the date `$match` commented out while `startDate`/`endDate` are computed but unused.

Read first: `invoiceController/summary.js:27-42`; compare `clientController/summary.js:38-49` (which *does* filter by date).

Plan: restore the date `$match` using the computed range; verify `type=month` excludes older invoices.

What could go wrong: the totalInvoice facet may be *intended* to be all-time — check the frontend dashboard's expectation before assuming. Reject reason: changing a facet the UI relies on being all-time.

Interview story potential: "found a validated-but-ignored parameter bug" (debugging, STAR-worthy).

---

## Ticket 10: Add a health check endpoint

Difficulty: Easy — 30 min — Skills: Express routing.

Story: As an operator, I want `GET /health` returning DB status.

Why good: enables any deploy/orchestration; no existing health check. Additive.

Read first: `app.js:39-46`; `mongoose.connection.readyState`.

Plan: add a small router `GET /health` → `{ status:'ok', db: mongoose.connection.readyState }`, mounted before the 404 handler and *without* auth.

Reject reason: putting it behind auth (defeats the purpose).

Interview story potential: "added the first observability primitive" (ops).

---

## Ticket 11: Add request logging middleware

Difficulty: Easy — 30 min — Skills: Express middleware.

Story: As a developer, I want request logs (method, path, status, latency).

Why good: zero visibility today. `morgan`/`pino-http` is a one-liner.

Read first: `app.js:19-33` (middleware order).

Plan: add `app.use(morgan('dev'))` after the parsers. Verify logs appear.

Reject reason: logging request bodies (may include passwords on `/login`).

Interview story potential: ops improvement.

---

## Ticket 12: Whitelist fields in the generic update

Difficulty: Medium — 45 min — Skills: security, refactoring.

Story: As a maintainer, I want generic update to reject unexpected fields.

Why good: fixes confirmed mass-assignment (`createCRUDController/update.js:9` spreads `req.body`).

Read first: `createCRUDController/update.js`; contrast `paymentController/update.js:48-56` (explicit `updates`).

Plan: this affects *all* generic entities — so scope carefully. Safer: add per-entity whitelists, or document why the generic one can't be blindly whitelisted (fields vary). Discuss in the PR.

Reject reason: breaking a legitimate field update for some entity by over-restricting. This is why it's Medium, not Easy.

Interview story potential: "closed a mass-assignment hole" (security, STAR-worthy).

---

## Ticket 13: Add unique compound index on invoice `{number, year}`

Difficulty: Medium — 45 min — Skills: indexes, data integrity.

Story: As a maintainer, I want the DB to reject duplicate invoice numbers per year.

Why good: closes the duplicate-number race at the DB layer (`Invoice.js:10-17`).

Read first: `Invoice.js:10-17`; the race in `InvoiceForm.jsx:40`.

Plan: add `invoiceSchema.index({ number: 1, year: 1 }, { unique: true })`. **Check existing data first** — if duplicates already exist, the index build fails; that's a finding to report, not force.

Reject reason: forcing the index on a DB with existing duplicates without a cleanup plan.

Interview story potential: "guarded invoice numbering with a unique index" (data integrity, STAR-worthy).

---

## Ticket 14: Prune expired tokens from `loggedSessions`

Difficulty: Medium — 1 hr — Skills: JWT, arrays.

Story: As a maintainer, I want `loggedSessions` not to grow forever.

Why good: fixes confirmed unbounded growth (`authUser.js:23`).

Read first: `authUser.js:15-29`; `isValidAuthToken.js:45-53`.

Plan: on login, filter out tokens that no longer `jwt.verify` before pushing; or cap the array length. Verify old tokens are pruned.

Reject reason: pruning *valid* tokens (logs out other devices unexpectedly).

Interview story potential: "fixed unbounded document growth" (reliability).

---

## Ticket 15: Escape user regex in search

Difficulty: Medium — 45 min — Skills: security (ReDoS), regex.

Story: As a maintainer, I want search input escaped so it can't be a ReDoS vector.

Why good: mitigates the *investigate*-level ReDoS in `search.js:17` and `paginatedList.js:15`.

Read first: `createCRUDController/search.js:13-19`; `paginatedList.js:14-16`.

Plan: escape regex metacharacters and cap `q` length before building the regex. Verify normal search still works.

Reject reason: breaking legitimate partial-match search behavior.

Interview story potential: "hardened search against ReDoS" (security).

---

## Ticket 16: Add a `.env.example` for the backend

Difficulty: Easy — 20 min — Skills: config, docs.

Story: As a new developer, I want to know which env vars the backend needs.

Why good: there's no committed template; the required vars are scattered in code. Pure docs, zero risk.

Read first: `server.js:14-19`, `isValidAuthToken.js:22`, `serverApiConfig.js`.

Plan: create `backend/.env.example` with `DATABASE=`, `JWT_SECRET=`, `PORT=8888`, `OPENAI_API_KEY=`, etc., with comments. No secrets.

Reject reason: committing real secrets.

Interview story potential: "improved onboarding" (collaboration).

---

## How to pick

Start with 1, 2, 7, 9 — they fix confirmed defects and teach the most about the codebase. Then 5, 13 (indexes/integrity), then 6, 12, 15 (security). Each is a real, mergeable contribution with a story attached.
