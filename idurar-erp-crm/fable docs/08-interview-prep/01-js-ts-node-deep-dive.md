# JS / TS / Node Deep-Dive Question Cards

15 cards. Each: the question as asked, what it's really testing, the repo anchor to reference, and junior/mid/senior answer layers. Practice aloud, 90 seconds each.

Note on TS: this repo is **plain JavaScript** — no TypeScript. Several cards use that fact ("what would types have caught here?") which is itself a strong talking point about the value of types.

---

## Q1: Walk me through what happens when this async Express handler is called and it throws.

Round: Node. What it's testing: async error propagation, why `try/catch` isn't enough.

Repo anchor: `backend/src/handlers/errorHandlers.js:9-32` (`catchErrors`) applied at `backend/src/routes/appRoutes/appApi.js:9-17`.

- **Junior:** "It's caught by try/catch." (But there's no try/catch in the controllers.)
- **Mid:** "Express 4 doesn't await handlers, so a rejected promise would be unhandled and the request would hang. `catchErrors` wraps each handler and `.catch()`es the rejection, mapping Mongoose `ValidationError` to 400 and everything else to 500."
- **Senior:** "…and Express 5 awaits handlers natively, making this wrapper unnecessary. I'd also stop echoing the raw `error` object to the client (`errorHandlers.js:22-28`) — that leaks internals."

Follow-up: "What if the controller calls `res.json()` and *then* throws?" (Headers already sent → `catchErrors` tries to send again → `ERR_HTTP_HEADERS_SENT`.)

---

## Q2: `0.1 + 0.2 !== 0.3`. Why, and how does this app handle money?

Round: JS. What it's testing: IEEE-754 awareness, practical money handling.

Repo anchor: `backend/src/helpers.js:47-60` and `frontend/src/utils/calculate.js`.

- **Junior:** "Floating point is imprecise."
- **Mid:** "Doubles can't represent 0.1 exactly, so sums drift. IDURAR routes all money math through `currency.js` (`calculate.add/sub/multiply`), which works in integer cents. Invoice totals use it at `invoiceController/create.js:31-38`."
- **Senior:** "But values are stored as plain `Number` in Mongo (`Invoice.js:108-111`), so precision is only protected *during* computation. And `paymentController/remove.js:39` uses raw `total - discount` instead of `calculate.sub`, reintroducing drift. I'd store integer cents in the DB or use `Decimal128`."

Follow-up: "Where else could raw float math sneak in?" (Any `+`/`-` on amounts outside the helper.)

---

## Q3: Explain the event loop using something this app does.

Round: Node. What it's testing: concurrency model, `Promise.all`, blocking.

Repo anchor: `backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:32-50` (parallel find + count) and `backend/src/controllers/pdfController/index.js:70-79` (blocking PDF).

- **Junior:** "Node is single-threaded and non-blocking."
- **Mid:** "The list endpoint fires the find and the count concurrently with `Promise.all` (`paginatedList.js:50`) — both I/O operations are in flight while the event loop is free. They're not truly parallel CPU work, they're overlapping I/O waits."
- **Senior:** "The PDF path is the counter-example: `html-pdf` does synchronous CPU-heavy rendering on the request thread (`pdfController/index.js:70-79`), which blocks the event loop and stalls *every other* request. That belongs in a worker thread or a queue."

Follow-up: "Microtask vs macrotask — where does `await` resume?" (Microtask queue, before the next timer/IO callback.)

---

## Q4: This login stores a JWT and also writes it to the database. Why both?

Round: Node/auth. What it's testing: stateless vs stateful sessions, revocation.

Repo anchor: `authUser.js:15-29` (sign + push), `isValidAuthToken.js:22-53` (verify + membership), `logout.js:11-18` (revoke).

- **Junior:** "For security."
- **Mid:** "Plain JWT can't be revoked before expiry. IDURAR signs a JWT *and* stores it in `AdminPassword.loggedSessions`; every request checks both the signature and that the token is still in the array (`isValidAuthToken.js:45-53`). Logout `$pull`s it, so the token stops working immediately."
- **Senior:** "The cost is a DB read per request — you've given back JWT's main benefit. And `loggedSessions` grows unbounded (`authUser.js:23`, no prune). I'd use a `tokenVersion` integer bumped on logout-all, or short-lived access tokens + refresh tokens, to get revocation without the array."

Follow-up: "Where's the token stored client-side and what's the risk?" (localStorage, `redux/auth/actions.js:20` → readable by any XSS.)

---

## Q5: What's a closure, and where does this codebase rely on one?

Round: JS. What it's testing: closures beyond the textbook.

Repo anchor: `createCRUDController/index.js:15-33` (handlers close over `Model`) and `createAuthMiddleware/index.js:7-35` (methods close over `userModel`).

- **Junior:** "A function that remembers its scope."
- **Mid:** "The CRUD factory resolves `mongoose.model(modelName)` once and returns handlers that *close over* that `Model` variable. Each handler is a closure carrying its own model — that's how one factory produces isolated controllers per entity."
- **Senior:** "Closures are also the stale-data trap in React: `DataTable.jsx:162-168`'s effect closes over props at mount with an empty dep array, so it captures the initial `entity`. Same language feature, opposite risk."

Follow-up: "How would generics/TS type this factory?" (`createCRUDController<T>(name): CrudController<T>`.)

---

## Q6: This code is untyped JS. Name three bugs TypeScript would have caught here.

Round: TS. What it's testing: understanding what types actually buy you.

Repo anchor: `frontend/src/router/routes.jsx:73-116` (references undefined `Quote`), `redux/erp/reducer.js:83` (magic-string `keyState`), the filename→route contract (`models/utils/index.js:22-47`).

- **Junior:** "Types catch typos."
- **Mid:** "1) `routes.jsx` renders `<Quote/>` with no import — TS would flag `Cannot find name 'Quote'` at compile time instead of a runtime `ReferenceError`. 2) `keyState` is a magic string; a typo silently writes a new state key — a union type `'list'|'create'|…` would reject it. 3) The API envelope has no shared type, so client and server can drift."
- **Senior:** "The deepest one is the filename→route→controller contract (`models/utils`): it's stringly-typed across the whole stack. TS alone can't fix a runtime glob, but a generated types file from the model registry would make renames a compile error instead of a broken frontend."

Follow-up: "`unknown` vs `any` — which for the API response and why?" (`unknown` forces a narrowing check; `any` disables safety.)

---

## Q7: Explain `Promise.all` vs sequential `await`, with a concrete tradeoff.

Round: JS. What it's testing: parallel vs serial async, error semantics.

Repo anchor: `paginatedList.js:32-50` (parallel), `isValidAuthToken.js:32-35` (parallel user+password fetch), vs `paymentController/create.js` (sequential dependent writes).

- **Junior:** "`Promise.all` runs them together."
- **Mid:** "Use `Promise.all` when calls are independent — the list endpoint fetches results and count simultaneously (`paginatedList.js:50`), halving latency. Use sequential `await` when one depends on the previous — the payment flow must create the payment *before* updating the invoice (`create.js:41` then `:66`)."
- **Senior:** "`Promise.all` rejects on the first failure and doesn't cancel the rest — for the two-write payment flow that's actually a hazard, because a partial failure leaves inconsistent state. `Promise.allSettled` when you want all outcomes; a transaction when you need atomicity."

Follow-up: "What does `Promise.all` do with the other promises when one rejects?" (They keep running; their results are discarded.)

---

## Q8: A user sends `total: 0.01` for an invoice with a $500 item. What happens?

Round: Node/security. What it's testing: server-authoritative computation.

Repo anchor: `invoiceController/create.js:30-43`.

- **Junior:** "It saves 0.01." (Wrong.)
- **Mid:** "Nothing bad — the server ignores the client's `total`. It recomputes `subTotal`, `taxTotal`, and `total` from `items` and overwrites `body` (`create.js:40-42`). Client math (`InvoiceForm.jsx:54-58`) is display-only."
- **Senior:** "Correct here, but the pattern is applied inconsistently: the invoice *number* IS taken from the client (`InvoiceForm.jsx:40`), which is exploitable/racy. The rule is 'never trust the client for anything money- or identity-relevant,' and I'd audit every write against it."

Follow-up: "What about `createdBy`?" (Set from `req.admin._id`, `create.js:48` — also correct.)

---

## Q9: What is NoSQL injection and is this app vulnerable?

Round: Node/security. What it's testing: input-trust in query construction.

Repo anchor: `paginatedList.js:19-29` (object guard) and `search.js:13-19` (regex from raw input).

- **Junior:** "SQL injection doesn't apply to Mongo."
- **Mid:** "It does — if you put a user-controlled *object* into a query, `{$gt:''}` matches everything. IDURAR guards the list `equal` param by rejecting non-primitive values (`paginatedList.js:21-27`). Good."
- **Senior:** "But `search.js:17-19` builds `new RegExp(req.query.q)` from raw input with no guard — a crafted pattern like `(a+)+$` is a ReDoS vector that can pin the event loop. I'd anchor/escape the input or cap its length. *Investigate*: I haven't proven a payload, but the shape is there."

Follow-up: "Why is `express.json()` + object params the real risk surface?" (Body/query can carry nested objects.)

---

## Q10: Explain `this` / arrow functions using a schema method here.

Round: JS. What it's testing: `this` binding.

Repo anchor: `backend/src/models/coreModels/AdminPassword.js:38-45` (`function` methods using `this`).

- **Junior:** "Arrow functions don't have their own `this`."
- **Mid:** "Exactly why Mongoose instance methods use `function` not `=>`: `generateHash`/`validPassword` (`AdminPassword.js:38-45`) rely on `this` being the document. An arrow function would capture the module scope and `this.password` would be undefined."
- **Senior:** "General rule: `function` when the caller sets `this` (methods, Mongoose hooks, event handlers); arrow when you want lexical `this` (React callbacks, `.map`). Getting this wrong in a `pre('save')` hook is a common silent bug."

Follow-up: "Where in the frontend does lexical `this` matter?" (It mostly doesn't — hooks/functional components avoid `this` entirely, which is part of why they're preferred.)

---

## Q11: Walk me through module resolution — how does `@/helpers` resolve on the backend?

Round: Node. What it's testing: module systems, path aliasing.

Repo anchor: `backend/package.json` (`_moduleAliases`, `module-alias/register` at `server.js:1`) vs frontend Vite alias (`frontend/vite.config.js` `resolve.alias`).

- **Junior:** "It's a relative import."
- **Mid:** "Backend is CommonJS; `@` maps to `src` via `module-alias` registered at the top of `server.js:1`. Frontend is ESM (`\"type\":\"module\"`); the same `@` alias is resolved by Vite at build time (`vite.config.js`). Two different mechanisms, same ergonomics."
- **Senior:** "The split matters: `module-alias` is a runtime hack that rewrites `require`, so tooling (jest, ts) needs separate config to understand it. A cleaner setup uses Node's native `imports` field or subpath imports. It's also why you can't share code directly between FE and BE despite the alias looking identical."

Follow-up: "CommonJS vs ESM — one concrete incompatibility?" (`require` is synchronous; `import` is async/hoisted; no `__dirname` in ESM.)

---

## Q12: How does this app paginate, and where does that approach break?

Round: Node/data. What it's testing: offset vs cursor pagination.

Repo anchor: `paginatedList.js:1-8, 37-39`.

- **Junior:** "It uses page and limit."
- **Mid:** "Offset pagination: `skip = page*limit - limit`, then `.skip().limit()` (`paginatedList.js:4,37-38`). Simple and supports jump-to-page."
- **Senior:** "It degrades on deep pages — `skip(100000)` still scans and discards 100k docs, and it can skip/duplicate rows if data changes between page loads. For large or fast-changing data I'd use keyset/cursor pagination (`find({_id: {$gt: lastId}}).limit()`). Also `page` arrives as a string and is used in arithmetic (`:2`) — works via coercion but is fragile."

Follow-up: "Why is `countDocuments` on every page a cost?" (Full count scan; cache it or use estimated count.)

---

## Q13: `req.query.page` is a string but used in math. Walk me through coercion.

Round: JS. What it's testing: type coercion pitfalls.

Repo anchor: `paginatedList.js:2-4` (`page` used as `page*limit-limit`), `:3` (`parseInt(req.query.items)`).

- **Junior:** "JS converts it automatically."
- **Mid:** "`items` is explicitly `parseInt`'d (`:3`), but `page` isn't — `'2' * 10 - 10` coerces `'2'` to `2` via the `*` operator, so it happens to work. `+` would've been string concatenation instead."
- **Senior:** "Relying on implicit coercion is a latent bug: `page='2abc'` → `NaN` → `skip(NaN)` throws or returns nothing. And `sortValue` from the query (`:6`) goes straight into `.sort()`. I'd validate/parse query params at the boundary (Joi/zod) rather than trust coercion."

Follow-up: "`==` vs `===` — where does the app rely on loose equality?" (`data.success === true` uses strict — good practice throughout the thunks.)

---

## Q14: What does `mongoose-autopopulate` do and what's the hidden cost?

Round: Node/data. What it's testing: ORM magic, N+1, over-fetching.

Repo anchor: `Invoice.js:31-36, 178` and `Payment.js:14-25, 56` (autopopulate on refs).

- **Junior:** "It fills in the referenced document."
- **Mid:** "Marking a ref `autopopulate:true` (e.g. `Invoice.client`, `Payment.invoice`) makes Mongoose automatically join it on every query via the plugin (`Invoice.js:178`). Convenient — the frontend gets `invoice.client.name` for free."
- **Senior:** "The cost is invisible over-fetching and cascading populates: `Payment` autopopulates `invoice`, which autopopulates its `client` — a single payment read triggers a chain of extra queries. On a list of 100 payments that's a hidden N+1. I'd populate explicitly per query where needed (like `read.js:11` does with `.populate('createdBy','name')`) instead of globally."

Follow-up: "How would you detect this in prod?" (Mongoose query logging / DB profiler.)

---

## Q15: The invoice number comes from the client. Design the race and the fix.

Round: Node/concurrency. What it's testing: atomicity, race conditions.

Repo anchor: `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:40` (`last_invoice_number + 1`), `invoiceController/create.js:62-64` (un-awaited `$inc`), `middlewares/settings/increaseBySettingKey.js:11-20`.

- **Junior:** "Each invoice gets the next number."
- **Mid:** "The number is read from a setting on the client and incremented there; the server bumps the counter separately and fire-and-forget (`create.js:62`, no `await`). Two users loading the form at the same time both see `N+1` and both submit `N+1` → duplicate."
- **Senior:** "Fix: allocate atomically on the server — `findOneAndUpdate({key},{ $inc:{value:1}},{new:true})` returns the reserved number, use that for the invoice. Or accept the race and add a unique compound index `{number, year}` so the DB rejects the second one and the client retries. The current code has neither, so duplicates are silently possible."

Follow-up: "Why doesn't the un-awaited `$inc` even guarantee the counter moves?" (Errors are swallowed, `increaseBySettingKey.js:29-31`, and the response can return first.)

---

## Coverage map

Repo-anchored: Q1–Q15 (15/15 anchored). Concepts covered: async error handling, floating-point money, event loop, JWT/sessions, closures, TS value, `Promise.all`, server-authoritative computation, NoSQL/ReDoS injection, `this`/arrows, module systems, pagination, coercion, ORM populate, concurrency/atomicity.
