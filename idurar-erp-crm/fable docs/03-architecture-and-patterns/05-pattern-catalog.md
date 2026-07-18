# Pattern Catalog

Fourteen patterns this codebase actually uses, each with the general shape, real anchors, why it works here, how it fails, and the interview question it answers. The goal is **recognition** — being able to spot the same shape in any repo and name it — not memorizing IDURAR trivia.

---

## Pattern: Controller Factory (parameterized module)

Problem it solves: writing identical CRUD handlers for every model.

General shape: a function takes a model name and returns an object of handlers closed over that model.

Real example: `backend/src/controllers/middlewaresControllers/createCRUDController/index.js:15-33` — `createCRUDController(modelName)` returns `{create, read, update, delete, list, …}`.

Second example: `backend/src/controllers/middlewaresControllers/createAuthMiddleware/index.js:7-35` — `createAuthMiddleware(userModel)` returns `{login, logout, isValidAuthToken, …}`.

Why this implementation works: the model is resolved once via `mongoose.model(modelName)` and captured in closures, so each handler is a thin adapter over Mongoose. New models get full CRUD for free.

Failure modes: the factory throws if the model isn't registered (`index.js:16-18`), so model-load order matters (`server.js:28-32` globs models before `app.js` runs). Handlers are generic, so anything model-specific (invoice totals) needs an override, and overrides are easy to forget.

Use it when: you have many entities with near-identical operations. Avoid it when: entities diverge enough that the factory becomes a mess of `if (entity === …)` — which is already creeping in at `appApi.js:19-25`.

Interview angle: "How would you avoid writing the same CRUD 20 times?" — factory + escape hatch. See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: find the third factory in the codebase (hint: uploads, `singleStorageUpload.js:7-56`) and describe what it's parameterized on.

---

## Pattern: Convention over Configuration (filename as contract)

Problem it solves: manually registering every model, route, and controller.

General shape: derive wiring from the filesystem instead of a config file.

Real example: `backend/src/models/utils/index.js:22-47` turns each file in `models/appModels/` into an `{entity, modelName, controllerName}` route descriptor.

Second example: `backend/src/controllers/appControllers/index.js:7-10` globs controller directories to discover custom controllers.

Why this implementation works: adding `Foo.js` gives you `/api/foo/*` with zero registration. Fast for a CRUD app with a fixed shape.

Failure modes: the filename *is* an implicit contract with no compile-time check. Renaming `Invoice.js` silently renames the API path and breaks the frontend's hardcoded `entity: 'invoice'`. This is the flip side of the trimmed models causing the frontend's dangling `/quote` route.

Use it when: the mapping is truly uniform and rarely renamed. Avoid it when: you need explicit, greppable route definitions or type-safe client/server contracts.

Interview angle: "Convention over configuration — pros and cons?" Speed vs implicit coupling and no static verification.

Drill: trace what breaks, in order, if you rename `Payment.js` to `Payments.js`. (Route becomes `/payments`, `paymentController` lookup misses, custom controller un-wires.)

---

## Pattern: Higher-Order Error Wrapper (`catchErrors`)

Problem it solves: repeating `try/catch` in every async Express handler.

General shape: wrap a handler so rejected promises are funneled to one place.

Real example: `backend/src/handlers/errorHandlers.js:9-32` — `catchErrors(fn)` returns a handler that calls `fn(...).catch(...)`.

Second example: applied at `backend/src/routes/appRoutes/appApi.js:9-17` and `coreAuth.js:8-13` on every route.

Why this implementation works: Express 4 doesn't await handlers, so an unhandled async rejection would otherwise hang the request. This wrapper converts a rejection into a 400 (Mongoose `ValidationError`) or 500 (`errorHandlers.js:12-28`).

Failure modes: it only catches what the handler *throws or rejects*. Controllers that `return res.status(202).json(...)` for a business failure (`paymentController/create.js:11-17`) bypass it, producing inconsistent status codes across the API. Also every error 500 leaks `error` into the JSON body (`errorHandlers.js:22-28`, `productionErrors:71-77`) — an information-disclosure smell.

Use it when: async handlers in a framework without native promise support. Avoid it when: your framework already awaits handlers (Express 5, Fastify).

Interview angle: "How do you handle errors across many async routes?" — centralize with a wrapper; mention Express 5 awaits natively.

Drill: list every distinct HTTP status the payment create path can return and whether each goes through `catchErrors`.

---

## Pattern: Soft Delete (`removed` flag)

Problem it solves: deleting records without losing history or breaking references.

General shape: never `deleteOne`; set a boolean and filter it out on reads.

Real example: `backend/src/controllers/middlewaresControllers/createCRUDController/remove.js:3-15` sets `removed: true`; every read filters `removed: false` (`paginatedList.js:33`, `read.js:8`).

Second example: cascading soft delete — deleting an invoice soft-deletes its payments (`invoiceController/remove.js:26-29`, `updateMany … removed:true`).

Why this implementation works: preserves audit trail and referential sanity; the filter is applied consistently in the generic handlers.

Failure modes: any query that forgets `removed:false` sees ghosts — and the download handler does exactly that (`downloadPdf.js:9-11` has no `removed` filter, so deleted invoices remain downloadable). Soft-deleted rows also accumulate forever (no purge job).

Use it when: you need recoverability or audit. Avoid it when: legal deletion (GDPR erasure) is required — then soft delete is a liability.

Interview angle: "Soft vs hard delete?" Recoverability and referential integrity vs the discipline cost of never forgetting the filter (point at the download-handler miss as the exact failure).

Drill: grep for every `findOne(`/`find(` in `backend/src` and mark which include `removed`. The misses are your risk list.

---

## Pattern: Denormalized Aggregate (`credit` / `paymentStatus` on Invoice)

Problem it solves: showing an invoice's paid-total and status without aggregating payments on every read.

General shape: cache a derived value on the parent; update it on every child mutation.

Real example: `Invoice.credit` and `Invoice.paymentStatus` (`Invoice.js:118-136`) maintained in `paymentController/create.js:66-77`, `update.js:66-77`, `remove.js:45-59`.

Second example: `Invoice.subTotal/taxTotal/total` cached from `items` (`invoiceController/create.js:40-43`).

Why this implementation works: list and read endpoints stay cheap — no per-row aggregation.

Failure modes: the cached value can drift from the source of truth if any mutation path forgets to update it, or if a crash interrupts the multi-write sequence (Flow 4). Status math is duplicated across three files and can diverge.

Use it when: reads vastly outnumber writes and recomputation is expensive. Avoid it when: you can afford to compute on read, or you can't guarantee every writer updates the cache.

Interview angle: "How do you keep denormalized/derived data consistent?" — single writer, transactions, or periodic reconciliation. This is central to [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: write a reconciliation query that recomputes `credit` for every invoice from its payments and reports mismatches.

---

## Pattern: Generic Reducer keyed by action-slice (`keyState`)

Problem it solves: writing a separate reducer for list/create/read/update/delete on every entity.

General shape: one reducer whose actions carry a `keyState` naming which sub-slice to update.

Real example: `frontend/src/redux/erp/reducer.js:70-127` — `REQUEST_LOADING/SUCCESS/FAILED` all write to `state[keyState]`.

Second example: the matching action creators dispatch `keyState` per operation (`redux/erp/actions.js:39-67` for `list`).

Why this implementation works: one reducer serves invoices, payments, clients, and any future entity. `createSelector` slices (`selectors.js:5-22`) memoize reads.

Failure modes: `keyState` is a magic string — a typo silently writes to a new key with no error. There's no type safety (no TS), so the shape of each slice is only enforced by `INITIAL_STATE` (`reducer.js:3-68`). State is per-entity-instance but the store is global, so two entity tables can stomp each other (the `resetState` on mount at `ErpPanelModule/index.jsx:21-27` is the workaround).

Use it when: many resources share the same request lifecycle. Avoid it when: slices diverge, or you'd benefit from RTK Query / a data-fetching library that does this for you.

Interview angle: "How do you manage server state in Redux without duplicating reducers?" — and "why might RTK Query replace this entirely?"

Drill: introduce a typo `keyStat: 'list'` in the list thunk and predict the UI symptom (spinner never resolves; `selectListItems` reads stale slice).

---

## Pattern: Thunk-based async action (Redux Toolkit, manual)

Problem it solves: sequencing loading→request→success/failed around an async call.

General shape: an action creator returns `async (dispatch) => { dispatch(loading); const data = await api(); dispatch(data.success ? success : failed) }`.

Real example: `frontend/src/redux/erp/actions.js:36-68` (`list`), `98-126` (`recordPayment`).

Second example: `frontend/src/redux/auth/actions.js:5-31` (`login`).

Why this implementation works: predictable three-phase lifecycle the generic reducer understands; the `data.success` envelope makes success/failure explicit.

Failure modes: every thunk hand-rolls the same boilerplate (RTK's `createAsyncThunk` exists to eliminate it). Errors are swallowed into `data.success === false` by the request layer, so thunks can't distinguish a network failure from a 404. No cancellation despite the unused `AbortController` (`DataTable.jsx:162-168`).

Use it when: a small, explicit async layer. Avoid it when: you'd benefit from `createAsyncThunk` or a query cache.

Interview angle: "Walk me through an async Redux flow" and "how would you modernize it?" (`createAsyncThunk`, RTK Query).

Drill: rewrite the `list` thunk using `createAsyncThunk` and note what boilerplate disappears.

---

## Pattern: Config-driven component (DataTable)

Problem it solves: rebuilding a table/CRUD screen per entity.

General shape: a component takes a `config` object (columns, entity, labels) and renders generically.

Real example: `frontend/src/modules/ErpPanelModule/DataTable.jsx:41-215` consumes `config.dataTableColumns`, `config.entity`, `searchConfig`.

Second example: each page builds that config — `frontend/src/pages/Invoice/index.jsx:21-102`.

Why this implementation works: one table implementation, N thin page configs. Columns, actions, and search are declarative.

Failure modes: config sprawl — the table hardcodes the action menu (`DataTable.jsx:54-80`) and knows about invoice-specific `handleRecordPayment` (`:102-105`), leaking entity specifics into the "generic" component. The download URL is built by string convention (`:93-95`).

Use it when: screens share layout and differ only in data. Avoid it when: per-entity screens diverge — the config grows conditionals.

Interview angle: "How do you build reusable UI without copy-paste?" — configuration vs composition tradeoffs.

Drill: identify every place `DataTable` special-cases invoices and propose how to inject that via `config`/`extra` instead.

---

## Pattern: Request envelope (`{success, result, message}`)

Problem it solves: a uniform response contract the client can branch on.

General shape: every endpoint returns the same top-level shape regardless of payload.

Real example: pervasive — `paginatedList.js:57-71`, `authUser.js:40-53`, `errorHandlers.js:12-28`.

Second example: the client keys off it everywhere — `redux/erp/actions.js:47` (`if (data.success === true)`).

Why this implementation works: the client never inspects HTTP status directly for business logic; `data.success` is the single branch. `successHandler`/`errorHandler` (`frontend/src/request/`) centralize notifications.

Failure modes: HTTP status and `success` can disagree (a 202 with `success:false` at `paymentController/create.js:11-17`), so status-based tooling (retries, monitoring) misreads outcomes. `error` objects are echoed to the client (info disclosure).

Use it when: you want a consistent client contract. Avoid it when: you rely on HTTP semantics (caching, REST tooling) — then align status with outcome.

Interview angle: "Envelope vs bare REST with status codes?" Consistency and easy client branching vs losing HTTP semantics.

Drill: list three endpoints where HTTP status and `success` disagree.

---

## Pattern: Server-authoritative computation (trust no client math)

Problem it solves: clients can lie about totals, prices, ownership.

General shape: recompute anything security- or money-relevant on the server; overwrite the client's value.

Real example: invoice totals recomputed and overwritten (`invoiceController/create.js:30-43`); `createdBy` set from `req.admin._id`, not the body (`create.js:48`); payment `maxAmount` re-derived server-side (`paymentController/create.js:30`).

Second example: `paymentStatus` derived server-side, never taken from the client (`create.js:45-47`).

Why this implementation works: the client form's math (`InvoiceForm.jsx:54-58`) is display-only; the server is the source of truth.

Failure modes: none in principle — this is the correct pattern. The gap is that it's applied inconsistently (invoice number *is* taken from the client, Flow 3).

Use it when: always, for money/auth/ownership. Avoid it when: never, for those.

Interview angle: "A user tampers with the price in the request — what stops them?" This is a top-tier answer with two concrete anchors.

Drill: send a `create` request with `total: 0.01` and a real item; confirm from `create.js:35-43` that the server overwrites it.

---

## Pattern: Dual-token session (stateless JWT + server allow-list)

Problem it solves: JWT can't be revoked; server sessions cost a lookup.

General shape: sign a JWT *and* store it in a per-user allow-list; verify both.

Real example: sign + push (`authUser.js:15-29`), verify signature + membership (`isValidAuthToken.js:22-53`), revoke via `$pull` on logout (`logout.js:11-18`).

Second example: none (single auth surface).

Why this implementation works: you get JWT's self-contained claims plus real revocation (logout, force-logout-all).

Failure modes: a DB read per request (the cost you were avoiding with JWT), and `loggedSessions` grows unbounded (`AdminPassword.js:30-33`) — expired tokens are never pruned.

Use it when: you need revocation but like JWT claims. Avoid it when: you can use short-lived access + refresh tokens instead (usually the better design).

Interview angle: "Stateless vs stateful auth" — this repo is the hybrid; you can weigh it precisely. See [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md).

Drill: design a `tokenVersion` integer scheme that achieves revocation without an unbounded array.

---

## Pattern: Decimal-safe money math (avoid float `+`)

Problem it solves: `0.1 + 0.2 !== 0.3` in IEEE-754 floats.

General shape: route all money arithmetic through a library.

Real example: `backend/src/helpers.js:47-60` wraps `currency.js`; used in `invoiceController/create.js:31-38` and `paymentController/create.js:30,60-64`.

Second example: frontend mirror `frontend/src/utils/calculate.js` used in `CreateItem.jsx:63-71`.

Why this implementation works: `currency.js` works in integer cents internally, avoiding float drift on totals.

Failure modes: the DB stores plain `Number` (`Invoice.js:108-111`), so once values leave `currency.js` they're floats again; mixing raw `+` anywhere (e.g. `paymentController/remove.js:39` uses raw `total - discount`) reintroduces drift. Duplicated FE/BE implementations can diverge.

Use it when: any currency handling. Avoid it when: never skip it for money.

Interview angle: "Why is `0.1 + 0.2` a problem and how do you handle money in JS?" — integer cents or a decimal library. See [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md).

Drill: find the one payment path that uses raw `-` instead of `calculate.sub` (`remove.js:39`) and explain the risk.

---

## Pattern: Lazy routes + Suspense (code splitting)

Problem it solves: shipping the whole app in one bundle.

General shape: `lazy(() => import(...))` each route, wrapped in `<Suspense fallback>`.

Real example: `frontend/src/router/routes.jsx:5-26` lazy-loads pages; `RootApp.jsx` and `IdurarOs.jsx:11` wrap in `Suspense`.

Second example: `IdurarOs.jsx:11-20` lazy-loads the entire authed app shell so the login bundle stays tiny.

Why this implementation works: unauthenticated users download only the auth bundle; each route is a separate chunk.

Failure modes: a lazy import that references a deleted module (the removed `Quote`) throws at render; here the imports were removed but the JSX references remain (`routes.jsx:73-116`), so the route table itself is broken. No error boundary catches a failed chunk load.

Use it when: route-level splitting on a multi-page SPA. Avoid it when: tiny apps where the overhead isn't worth it.

Interview angle: "How do you reduce initial bundle size in React?" — route-level `lazy` + `Suspense`, plus an error boundary for chunk failures.

Drill: add an error boundary around `<AppRouter/>` and describe what the user sees when a chunk 404s.

---

## Pattern: Path-traversal guard on file serving

Problem it solves: `../../etc/passwd`-style escapes when serving user-pathed files.

General shape: resolve the absolute path, then assert it stays inside a trusted root before serving.

Real example: `backend/src/routes/coreRoutes/corePublicRouter.js:13-29` joins decoded segments, computes `absolutePath`, and rejects if `!isPathInside(absolutePath, rootDir)`.

Second example: the helper itself, `backend/src/utils/is-path-inside.js`.

Why this implementation works: even if a segment contains `..`, the containment check fails and returns 400 instead of leaking a file.

Failure modes: the sibling `/download` route (`coreDownloadRouter.js`) does *not* apply this guard and instead trusts a filename-derived id — different risk profile (no traversal, but no auth). Decoding order matters; double-encoding bugs are common in this pattern.

Use it when: any endpoint serves a filesystem path derived from user input. Avoid it when: you can serve by opaque id + DB lookup instead (safer).

Interview angle: "How do you prevent directory traversal?" — canonicalize then containment-check; show `is-path-inside`. See [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).

Drill: request `/public/..%2f..%2fpackage.json` and predict the result from `corePublicRouter.js:24-29`.

---

## Recognition summary

| Pattern | One-line tell | Anchor |
| --- | --- | --- |
| Controller Factory | `create...Controller(name)` returns handlers | `createCRUDController/index.js:15` |
| Convention over Config | routes derived from filenames | `models/utils/index.js:22` |
| Error Wrapper | `catchErrors(fn)` around routes | `errorHandlers.js:9` |
| Soft Delete | `removed:true` + `removed:false` filters | `remove.js:3` |
| Denormalized Aggregate | `credit`/`paymentStatus` on Invoice | `Invoice.js:118` |
| Generic Reducer | `state[keyState]` switch | `erp/reducer.js:83` |
| Manual Thunk | `dispatch(loading)…await…dispatch(success)` | `erp/actions.js:36` |
| Config-driven Component | `<DataTable config={...}/>` | `DataTable.jsx:41` |
| Request Envelope | `{success, result, message}` everywhere | `paginatedList.js:57` |
| Server-authoritative | server overwrites client totals | `invoiceController/create.js:30` |
| Dual-token Session | JWT + `loggedSessions` allow-list | `isValidAuthToken.js:45` |
| Decimal-safe Money | `calculate.add/sub/multiply` | `helpers.js:47` |
| Lazy + Suspense | `lazy(() => import())` | `routes.jsx:5` |
| Path-traversal Guard | `isPathInside(abs, root)` | `corePublicRouter.js:24` |
