# File Reading Order

25 files in the order that makes IDURAR click. Per file: why it matters, what to look for, what to ignore. Three paths — a **junior** builds a mental model, a **mid** can change safely, a **senior** critiques. Do the path that matches your goal; each is a superset of the one before.

## Junior path (files 1–12): build the model

| # | File | Why | Look for | Ignore |
| --- | --- | --- | --- | --- |
| 1 | `backend/src/server.js` | boot sequence | model glob (`:28-32`), Mongo connect (`:17`), port (`:36`) | the OpenAI key line |
| 2 | `backend/src/app.js` | whole server in 52 lines | the 5 router mounts and which have auth (`:39-43`) | commented fileUpload |
| 3 | `backend/src/routes/coreRoutes/coreAuth.js` | the public auth routes | login/logout/reset mapping | — |
| 4 | `backend/src/routes/appRoutes/appApi.js` | routes are *generated* | `routerApp` 9 routes (`:9-17`), `routesList.forEach` (`:28`) | quote/convert special-case |
| 5 | `backend/src/models/utils/index.js` | where entities come from | filename→entity/model/controller (`:22-47`) | the unused `constrollersList` |
| 6 | `backend/src/models/appModels/Invoice.js` | the central document | embedded `items`, `credit`, `paymentStatus`, refs | commented-out item fields |
| 7 | `backend/src/models/appModels/Payment.js` | the child that mutates the parent | refs to invoice+client, autopopulate | — |
| 8 | `backend/src/controllers/middlewaresControllers/createCRUDController/index.js` | the factory | the 9 methods it returns (`:21-31`) | — |
| 9 | `frontend/src/apps/IdurarOs.jsx` | the auth gate | `isLoggedIn ? ErpApp : AuthRouter` (`:60-71`) | commented network code |
| 10 | `frontend/src/router/routes.jsx` | the route table | the invoice/payment routes | the broken quote/tax entries (know they're broken) |
| 11 | `frontend/src/request/request.js` | every network call | `includeToken` (`:16-25`), CRUD methods | — |
| 12 | `frontend/src/pages/Invoice/index.jsx` | how a page is just config | `dataTableColumns`, `config` object | column styling |

At this point you can trace a list request from click to DB.

## Mid path (add files 13–20): change safely

| # | File | Why | Look for |
| --- | --- | --- | --- |
| 13 | `backend/src/controllers/appControllers/index.js` | custom-vs-generic resolution | `hasCustomControllers` fallback (`:16-33`) |
| 14 | `backend/src/controllers/appControllers/invoiceController/index.js` | how overrides are assembled | which methods are replaced (`:12-18`) |
| 15 | `backend/src/controllers/appControllers/invoiceController/create.js` | business logic lives here | server-authoritative totals (`:30-48`) |
| 16 | `backend/src/controllers/appControllers/paymentController/create.js` | the money invariant | the 3 writes, no transaction (`:41-77`) |
| 17 | `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js` | per-request auth | JWT verify + `loggedSessions` check (`:22-53`) |
| 18 | `frontend/src/redux/erp/actions.js` | async thunks | the loading→success/failed pattern (`:36-68`) |
| 19 | `frontend/src/redux/erp/reducer.js` | one reducer for all entities | `state[keyState]` switch (`:70-127`) |
| 20 | `frontend/src/modules/ErpPanelModule/DataTable.jsx` | the config-driven table | invoice special-casing (`:102-105`), effect deps (`:162-168`) |

Now you can add a field across the stack and know what you'll touch.

## Senior path (add files 21–25): critique and design

| # | File | Why | Look for |
| --- | --- | --- | --- |
| 21 | `backend/src/handlers/errorHandlers.js` | error contract | `catchErrors`, info leak (`:22-28`) |
| 22 | `backend/src/handlers/downloadHandler/downloadPdf.js` | the unauthenticated boundary | no `removed` filter (`:9-11`), regenerate cost |
| 23 | `backend/src/controllers/appControllers/clientController/summary.js` | the only real aggregation | `$facet`+`$lookup` (`:24-73`) |
| 24 | `backend/src/setup/setup.js` | seed + the broken references | requires missing models (`:54-67`) |
| 25 | `frontend/src/components/AutoCompleteAsync/index.jsx` | the trickiest component | debounce + async race (`:55-103`) |

Now you can write the architecture critique yourself and defend it.

## What to actively ignore across the repo

- The 40+ marketing README translations in `features/` — not code.
- `locale/translation/*` language files — mechanism matters, contents don't.
- Commented-out code (there's a lot: cookie auth in `authUser.js:31-39`, network detection in `IdurarOs.jsx:29-61`) — read once for intent, then ignore.
- `middlewares/inventory/` — appears vestigial; not wired into the flows.

## Drill

After the junior path, close the docs and answer without looking: (1) name the file that turns `Invoice.js` into the `/api/invoice` routes; (2) name the single file every API call passes through on the frontend. (Answers: `models/utils/index.js` and `request/request.js`.) If you can't, re-read files 4–5 and 11.
