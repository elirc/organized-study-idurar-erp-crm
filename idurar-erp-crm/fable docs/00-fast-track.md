# Fast Track — One Weekend

Goal: by Sunday night you can run the app, trace two end-to-end flows, make one safe change, and explain the architecture aloud as if in an interview.

## 1. Install and run (Saturday morning)

All commands **inferred** from the manifests and `INSTALLATION-INSTRUCTIONS.md` — they were not executed while writing this curriculum (no MongoDB instance was available). Marked accordingly.

| Step | Command | Where | Status |
| --- | --- | --- | --- |
| Backend deps | `npm install` | `backend/` | inferred |
| Create `.env` | `DATABASE=mongodb://localhost/idurar` + `JWT_SECRET=...` | `backend/.env` | inferred from `backend/src/server.js:14-17` and `isValidAuthToken.js:22` |
| Seed admin + settings | `npm run setup` | `backend/` | inferred — **expect a crash**, see below |
| Run API | `npm run dev` (nodemon, port 8888) | `backend/` | inferred from `backend/package.json` scripts and `backend/src/server.js:36` |
| Frontend deps | `npm install` | `frontend/` | inferred |
| Run UI | `npm run dev` (Vite, port 3000, proxies `/api` to 8888) | `frontend/` | inferred from `frontend/vite.config.js` |

Two known landmines in this copy (both confirmed by reading, not running):

1. **`npm run setup` will crash partway.** `backend/src/setup/setup.js:54-67` requires `../models/appModels/PaymentMode` and `.../Taxes`, which do not exist in `backend/src/models/appModels/` (only `Client.js`, `Invoice.js`, `Payment.js`). The admin (`admin@admin.com` / `admin123`, `setup.js:18-26`) and settings are created *before* the crash, so login may still work. Fixing the script cleanly is Ticket 14 in [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md).
2. **The frontend will throw `ReferenceError: Quote is not defined`.** `frontend/src/router/routes.jsx:73-87` and `:110-116` render `<Quote />`, `<PaymentMode />`, `<Taxes />` but the lazy imports for them were removed (only the components at `routes.jsx:5-26` exist). Deleting those route entries (and the matching nav links in `frontend/src/apps/Navigation/NavigationContainer.jsx`) is Ticket 1 — do it now; it is the "one small safe change" of this fast track.

## 2. The first 10 files to open, in order

1. `backend/src/app.js` — the whole server topology in 52 lines. Note which routers get `adminAuth.isValidAuthToken` (lines 39-43) and which don't.
2. `backend/src/routes/appRoutes/appApi.js` — routes are *generated*, not listed: nine REST-ish endpoints per entity (lines 8-26).
3. `backend/src/models/utils/index.js` — where the entity list comes from: file names in `models/appModels/` (lines 22-47). Add a model file, get an API.
4. `backend/src/controllers/middlewaresControllers/createCRUDController/index.js` — the factory that gives every model default handlers.
5. `backend/src/controllers/appControllers/index.js` — how custom controllers (invoice, payment, client) override the defaults (lines 16-33).
6. `backend/src/models/appModels/Invoice.js` — the central document: embedded `items`, denormalized `credit`, `paymentStatus` enum.
7. `frontend/src/apps/IdurarOs.jsx` — the auth gate: `isLoggedIn ? <ErpApp/> : <AuthRouter/>`.
8. `frontend/src/request/request.js` — every network call in the app funnels through this object.
9. `frontend/src/redux/erp/reducer.js` — one generic reducer keyed by `keyState` handles list/create/read/update/delete for every entity.
10. `frontend/src/modules/ErpPanelModule/DataTable.jsx` — the config-driven table each entity page reuses.

## 3. Trace two flows (Saturday afternoon)

Do these with the code open; write each step down. Full trace tables live in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md).

**Flow A — Login:** `Login page → redux/auth/actions.js login thunk (:5-31) → auth.service.js POST /api/login (:7-27) → backend coreAuth.js (:8) → login.js Joi check (:13-29) → authUser.js bcrypt compare + JWT sign (:5-21) → token pushed into AdminPassword.loggedSessions (:23-29) → frontend stores auth state in localStorage (actions.js:20)`.

Pause and predict before reading: where does the browser keep the token, and how does it reach the server on the next request? (Answer: localStorage → `request.js:16-25` sets a `Bearer` default header on axios.)

**Flow B — Record a payment against an invoice:** `invoice list row → DataTable.jsx handleRecordPayment (:102-105) → /invoice/pay/:id → RecordPaymentModule/index.jsx (reads invoice from list cache or refetches) → RecordPayment.jsx computes maxAmount (:27-32) → erp.recordPayment thunk (redux/erp/actions.js:98-126) → POST /api/payment/create → paymentController/create.js: overpay guard (:32-38), payment saved (:41), then invoice updated with $inc credit + new paymentStatus (:66-77)`.

Pause and predict: what happens if the server crashes between saving the payment and updating the invoice? (There is no transaction — this is the repo's best interview story. See [03-architecture-and-patterns/04-side-effects-async-and-reliability.md](03-architecture-and-patterns/04-side-effects-async-and-reliability.md).)

## 4. One test to run (Sunday)

There is no test suite (`backend/package.json` has no test script; no `*.test.*` files exist — verified via file inventory). Instead, run the lint gate the repo does have: `npm run lint` in `frontend/` (**inferred**). Then do the manual smoke test: log in, create a client, create an invoice for that client, record a partial payment, confirm the invoice row shows `partially`.

## 5. Teach-back (Sunday night)

Explain aloud, 5 minutes, no notes, as if an interviewer asked "walk me through a codebase you know well":

1. What the product does, in one sentence.
2. How a request flows from a React page to MongoDB and back.
3. The convention-over-configuration trick (model file → routes → controller).
4. One thing you'd fix first and why (you have at least two confirmed candidates from step 1).

Grade yourself with the rubric in [09-reference/learning-rubrics.md](09-reference/learning-rubrics.md).

## What the fast path skips

Settings/localization internals, PDF generation, file uploads, the aggregation pipelines in `summary` endpoints, security analysis, and all of the drills. Those live in modules 01–05.
