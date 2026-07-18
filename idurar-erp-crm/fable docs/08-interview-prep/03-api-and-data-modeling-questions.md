# API & Data Modeling Question Cards

13 cards on REST/RPC design, validation, pagination, auth, schema design, migrations, transactions, and caching — anchored to IDURAR's actual API and Mongoose schema. Practice aloud, 90 seconds each.

---

## Q1: Design the REST API for invoices. What does this app do?

Round: API. What it's testing: REST resource design.

Repo anchor: `backend/src/routes/appRoutes/appApi.js:8-26`.

- **Junior:** "GET, POST, PUT, DELETE."
- **Mid:** "IDURAR generates per-entity routes: `create` (POST), `read/:id` (GET), `update/:id` (PATCH), `delete/:id` (DELETE), plus `list`, `search`, `listAll`, `filter`, `summary` (GET). PATCH not PUT because updates are partial. Invoice/payment also get `mail` (`:19-21`)."
- **Senior:** "The paths aren't RESTful — `/invoice/create` instead of `POST /invoice`, `/invoice/delete/:id` instead of `DELETE /invoice/:id`. It works but ignores HTTP verb semantics, so you can't rely on method-based caching/routing. A verb-in-path API is a pragmatic choice; I'd note it as a contract decision, not an accident."

Follow-up: "Why PATCH over PUT here?" (Client sends a partial body; PUT implies full replacement.)

---

## Q2: How does the API validate input? Walk me through the layers.

Round: API. What it's testing: validation strategy, defense in depth.

Repo anchor: `invoiceController/schemaValidate.js:2-24` (Joi), `Invoice.js` (Mongoose `required`/`enum`), `paginatedList.js:19-29` (query guard).

- **Junior:** "It checks required fields."
- **Mid:** "Two layers: Joi validates the request body shape up front (`schemaValidate.js` — client is a string-or-object, items non-empty, each item has name/qty/price), and Mongoose enforces schema constraints on save (`required`, `enum` like `status`). Belt and suspenders."
- **Senior:** "The layers aren't consistent — generic CRUD (`createCRUDController/create.js`) has *no* Joi, only Mongoose, so different entities have different validation rigor. And query params (`sortBy`, `filter`) go largely unvalidated into the query (`paginatedList.js:6`). I'd centralize validation at the boundary with one schema library and validate query strings too."

Follow-up: "What status does a Joi failure return vs a Mongoose failure?" (Joi → 400 in the controller; Mongoose `ValidationError` → 400 via `catchErrors`.)

---

## Q3: Model an invoice with line items. Embed or reference?

Round: data. What it's testing: document modeling.

Repo anchor: `Invoice.js:51-95` (embedded items) vs `Invoice.js:126-131` (referenced payments).

- **Junior:** "Make an items table."
- **Mid:** "Embed line items — they have no identity outside the invoice, are always read with it, and are never queried alone (`Invoice.js:51-95`). Reference payments (`:126-131` stores payment ids) because payments are created/edited/listed independently. Embed the owned-and-bounded child; reference the independently-managed one."
- **Senior:** "Embedding items caps invoice document size (fine — invoices have few items) and gives atomic updates of the whole invoice. If items were unbounded or shared, embedding would blow the 16MB doc limit and duplicate data. The payment array is a reference *list* which can grow — worth watching, but payments-per-invoice is naturally bounded."

Follow-up: "When would you regret embedding items?" (If you needed to query 'all invoices containing product X' across the collection efficiently.)

---

## Q4: There's no per-user data scoping. Is that a bug?

Round: API/security. What it's testing: authorization vs authentication, IDOR.

Repo anchor: `paginatedList.js:32-41` (no ownership filter), `app.js:40-41` (auth only), `Admin.js:30-34` (single role).

- **Junior:** "It checks the token, so it's secure."
- **Mid:** "Authentication (who are you) is enforced — every `/api` route is behind `isValidAuthToken`. But there's no *authorization* scoping: any logged-in admin can read/update/delete any record; queries filter only `removed:false`, never `createdBy:me`. In a single-tenant, single-`owner`-role app that's acceptable by design."
- **Senior:** "The moment you add a second user or tenant, this becomes a broken-access-control / IDOR vulnerability — user A could `read/:id` user B's invoice. The fix is to inject an ownership/tenant filter into every query at the middleware layer, not per-controller, so it can't be forgotten. That's the #1 thing to change before multi-user."

Follow-up: "Why is fixing it per-controller dangerous?" (One forgotten filter = a hole; centralize it.)

---

## Q5: Recording a payment touches two documents with no transaction. Consequences?

Round: data. What it's testing: ACID, transactions, invariants.

Repo anchor: `paymentController/create.js:41` (create payment) then `:66-77` (update invoice).

- **Junior:** "It saves the payment and updates the invoice."
- **Mid:** "Two independent writes: `Payment.create`, then invoice `$inc credit` + `$set paymentStatus`. No transaction — if the process crashes between them, the payment exists but `invoice.credit` is stale. The invariant `credit === SUM(payments)` is broken with no auto-repair."
- **Senior:** "Fix: `mongoose.startSession()` + `session.withTransaction()` around both writes — requires a replica set (Mongo transactions don't run on standalone). Cheaper alternative: derive `credit` on read so there's a single source of truth. Also no idempotency key, so a retried request double-counts. This is the repo's best system-design story."

Follow-up: "Why don't transactions work on a standalone mongod?" (They need the oplog/replica-set consensus machinery.)

---

## Q6: How would you add a new field to the Invoice schema safely?

Round: data/migrations. What it's testing: schema evolution, backward compatibility.

Repo anchor: `Invoice.js` schema; no migration tooling in the repo.

- **Junior:** "Add it to the schema."
- **Mid:** "MongoDB is schemaless at the storage layer, so adding an optional field with a default (`Invoice.js` style) is backward-compatible — old docs just lack it and read as the default. No migration needed for additive optional fields."
- **Senior:** "The trap is *required* or *renamed* fields: old documents fail validation or read as undefined. For those you need a backfill migration script and a deploy order (add optional → backfill → make required). This repo has *no* migration framework, so schema changes are ad hoc — I'd add one (migrate-mongo) before any breaking change and note that `setup.js` is a seed, not a migration."

Follow-up: "How do you make a field required when old docs don't have it?" (Backfill first, then tighten the constraint in a later deploy.)

---

## Q7: What indexes does this app need and which are missing?

Round: data/performance. What it's testing: indexing, query patterns.

Repo anchor: schemas have almost no explicit indexes; queries at `paginatedList.js:32-39`, `login.js:31`, `isValidAuthToken.js:32-33`.

- **Junior:** "Index the id." (Mongo does `_id` automatically.)
- **Mid:** "Hot query paths need indexes: `Admin.email` (login lookup, `login.js:31`), `AdminPassword.user` (already `unique`, `AdminPassword.js:11`), `Invoice.client` and `Invoice.removed` (list/filter). Right now most aren't declared, so those are collection scans as data grows."
- **Senior:** "The list endpoint sorts by `enabled` desc by default (`paginatedList.js:6`) and filters `removed:false` — that's a compound index candidate `{removed:1, enabled:-1}`. And `{number, year}` should be a *unique* compound index to close the duplicate-invoice-number race. Indexes cost write throughput and storage, so I'd add them driven by real slow-query logs, not speculatively."

Follow-up: "How do you find the missing index in prod?" (`explain()` on the query, or the Atlas/DB profiler flagging COLLSCAN.)

---

## Q8: The search endpoint builds a regex from user input. Problems?

Round: API/security. What it's testing: injection, ReDoS.

Repo anchor: `createCRUDController/search.js:13-19`, `paginatedList.js:14-16`.

- **Junior:** "It searches by name."
- **Mid:** "It builds `new RegExp(req.query.q, 'i')` per field and `$or`s them (`search.js:17-19`). Case-insensitive substring search. Functional."
- **Senior:** "Two issues: unanchored `$regex` can't use an index (full scan), and an untrusted regex is a ReDoS vector — a pattern like `(a+)+$` can pin the single-threaded event loop and take down the server (*investigate*: shape is there, payload unproven). I'd escape the input to a literal, or use a text index (`$text`) for real search. Escaping also prevents users injecting regex metacharacters."

Follow-up: "Why can't `$regex` without an anchor use an index?" (A leading wildcard means every doc must be tested.)

---

## Q9: Design pagination for the invoice list. What does this use?

Round: API. What it's testing: pagination strategies.

Repo anchor: `paginatedList.js:1-8, 37-50`.

- **Junior:** "Page number and size."
- **Mid:** "Offset pagination: `skip((page-1)*limit).limit(limit)`, plus a `countDocuments` for total pages, run together in `Promise.all` (`:50`). Supports jump-to-page and a page count."
- **Senior:** "Offset degrades on deep pages (`skip` scans discarded rows) and can double/skip rows if data shifts between requests. For large/live data use keyset pagination (`_id > lastSeen`). Also `countDocuments` on every request is its own scan — cache it or use `estimatedDocumentCount` for large collections. The `page` param arrives as a string used in arithmetic (`:2`), fragile but works via coercion."

Follow-up: "Trade-off of keyset vs offset?" (Keyset is fast and stable but can't jump to an arbitrary page.)

---

## Q10: Why does the API wrap everything in `{success, result, message}`?

Round: API. What it's testing: response contract design.

Repo anchor: `paginatedList.js:57-71`, `errorHandlers.js:12-28`, consumed at `redux/erp/actions.js:47`.

- **Junior:** "To return data."
- **Mid:** "A uniform envelope so the client branches on `data.success` regardless of endpoint (`actions.js:47`), and `message` gives user-facing text. `successHandler`/`errorHandler` centralize notifications off it."
- **Senior:** "The cost is decoupling HTTP status from outcome — `paymentController/create.js:11-17` returns 202 with `success:false` for a validation failure, so HTTP-status-based tooling (retries, alerting, CDN caching) misreads it. And error objects are echoed into the body (`errorHandlers.js:22-28`), leaking internals. I'd align status codes with outcomes and strip error internals in prod."

Follow-up: "Name three endpoints where status and `success` disagree." (Payment create/update/remove all use 202 for failures.)

---

## Q11: How would you add caching to the dashboard summary?

Round: API/performance. What it's testing: caching strategy, invalidation.

Repo anchor: `invoiceController/summary.js:33-125` (aggregation on every request); `node-cache` is a dependency but check if used.

- **Junior:** "Cache the result."
- **Mid:** "The summary runs a multi-stage aggregation on every dashboard load (`summary.js:33`). It changes only when invoices change, so cache the result (in-memory `node-cache` or Redis) keyed by `type`, TTL of a minute or two — the dashboard tolerates slight staleness."
- **Senior:** "The hard part is invalidation: bust the cache on any invoice create/update/delete, or accept TTL staleness. For a single-node app `node-cache` is fine; multi-node needs Redis so all instances share/invalidate. I'd start with a short TTL (simplest, good enough for a dashboard) and only add write-through invalidation if staleness actually matters."

Follow-up: "TTL vs write-through invalidation — when each?" (TTL for tolerable staleness/simplicity; write-through when reads must reflect the last write immediately.)

---

## Q12: The download endpoint has no auth. How would you secure it without breaking `window.open`?

Round: API/security. What it's testing: auth on non-XHR requests.

Repo anchor: `app.js:42` (no middleware), `DataTable.jsx:93-95` (`window.open`), `downloadPdf.js:6-13`.

- **Junior:** "Add the auth middleware."
- **Mid:** "You can't just add `isValidAuthToken` — `window.open` (`DataTable.jsx:94`) can't send an `Authorization` header, so the download would 401. That's *why* it's public. Options: a signed, short-lived URL (HMAC of id+expiry), or a one-time token query param the server validates."
- **Senior:** "I'd issue a signed URL from an authenticated endpoint (`GET /invoice/:id/pdf-url` returns `…/download/...?sig=…&exp=…`), verify the signature in the download route, and drop the raw path. Also add the missing `removed:false` filter (`downloadPdf.js:9-11`) so deleted invoices can't be fetched, and move generation to a cache so it isn't re-rendered every hit. Rank by blast radius: signed URL is the smallest, most standard change."

Follow-up: "Why is an ObjectId in the URL not enough?" (Obscurity isn't authorization; leaked/logged URLs stay valid forever.)

---

## Q13: `createdBy` comes from the token, not the request body. Why does that matter?

Round: API/security. What it's testing: trust boundaries, mass assignment.

Repo anchor: `invoiceController/create.js:48`, `paymentController/create.js:39`, `update.js:1-9` (spreads `req.body`).

- **Junior:** "It records who made it."
- **Mid:** "Ownership is stamped server-side from `req.admin._id` (`create.js:48`), never from the client body. If it came from the body, a user could forge `createdBy` to impersonate another admin. Correct pattern."
- **Senior:** "The generic `update.js` is the counter-risk: it does `Model.findOneAndUpdate(id, req.body)` (`createCRUDController/update.js:4-9`), spreading the *entire* body into the update — a mass-assignment hazard. A client could set fields it shouldn't (`removed`, `createdBy`, `credit`). I'd whitelist updatable fields instead of trusting the whole body. The invoice/payment overrides are safer because they reconstruct the update explicitly."

Follow-up: "What's mass assignment and where's it live here?" (Blindly binding request body to a model; `createCRUDController/update.js:9`.)

---

## Coverage map

Repo-anchored: Q1–Q13 (13/13). Concepts: REST design, validation layers, embed-vs-reference, authz/IDOR, transactions/invariants, migrations, indexing, regex injection/ReDoS, pagination, response envelope, caching + invalidation, auth on downloads, mass assignment.
