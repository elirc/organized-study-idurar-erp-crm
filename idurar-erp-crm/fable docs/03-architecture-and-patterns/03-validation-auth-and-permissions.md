# Validation, Auth & Permissions

## Validation layers (map)

| Layer | Where | Enforces | Coverage |
| --- | --- | --- | --- |
| Client-side form | `InvoiceForm.jsx` Ant `rules`, `ItemRow.jsx:59-68` | UX-only (required, patterns) | never trusted by server |
| Joi (boundary) | `invoiceController/schemaValidate.js:2-24`, `login.js:13-20` | request shape before DB work | invoice, payment, auth only |
| Query guard | `paginatedList.js:19-29` | rejects object `equal` (NoSQL injection) | list endpoint |
| Mongoose (persistence) | schemas: `required`, `enum`, `uppercase` | shape on save/update | all models |
| Path guard | `corePublicRouter.js:24-29` | directory traversal | `/public` only |

**The gap:** generic CRUD entities have *no* Joi — only Mongoose validation. So an invoice gets two validation layers; a hypothetical generic entity gets one. Validation rigor is inconsistent across the API. And query params like `sortBy`/`filter` flow largely unvalidated into queries (`paginatedList.js:6`).

## Authentication vs authorization

These are different and the distinction is a top interview topic.

**Authentication (who are you):** fully enforced. Every `/api` route except login/reset sits behind `isValidAuthToken` (`app.js:40-41`). The check is a *dual-token* design (`isValidAuthToken.js:22-53`): verify the JWT signature **and** confirm the token is still in the user's `loggedSessions` array. The second check is what enables revocation — logout `$pull`s the token (`logout.js:11-18`) and it stops working immediately, something pure JWT can't do.

**Authorization (what may you do):** almost none. Once authenticated, any admin can read/update/delete *any* record. Queries filter `removed:false` but never `createdBy: req.admin._id` (`paginatedList.js:32-41`, `read.js:3-6`). There is exactly one role (`Admin.role` enum `['owner']`, `Admin.js:30-34`).

## Is the missing authorization a bug?

**In this app, no — by design.** IDURAR is single-tenant with one owner role. There's no second user to isolate from, so per-record scoping would be ceremony. Stamping `createdBy` from the token (`invoiceController/create.js:48`) is done for *attribution*, not access control.

**The moment you add a second user or tenant, it becomes a critical IDOR/broken-access-control vulnerability.** User A could `GET /api/invoice/read/<B's id>` and read B's invoice. This is the #1 change required before multi-user, and the right fix is architectural: inject the ownership/tenant filter in shared middleware or a base query so it can't be forgotten per-controller. A per-controller fix is dangerous because one omission is a hole.

## What a junior misses vs what a senior checks

| Junior assumes | Senior verifies |
| --- | --- |
| "token check = secure" | *which* routes have the middleware — `/download` doesn't (`app.js:42`) |
| "the JWT proves identity" | the app also checks `loggedSessions`, so a valid JWT can still be rejected |
| "validation is done" | *which* endpoints have Joi vs only Mongoose |
| "createdBy scopes access" | it doesn't — no query filters by it |
| "ObjectId is unguessable = safe" | obscurity ≠ authorization (the download route) |

## Password handling

- Stored as bcrypt hash of `salt + password`, where `salt` is a random `shortid` (`authUser.js:5`, `setup.js:16-18`). bcrypt already salts internally, so the extra prepended salt adds little — a talking point, not a vulnerability.
- Passwords live in a separate `AdminPassword` collection so they don't serialize with the user (`authUser.js:40-53` returns the user sans password).
- Reset flow uses `resetToken`/`emailToken` fields (`AdminPassword.js:20-21`).

## Token storage (client)

The JWT is stored in `localStorage` (`redux/auth/actions.js:20`) and attached as a `Bearer` header (`request.js:16-25`). localStorage is XSS-readable — a single injected script exfiltrates the token. The commented-out cookie code (`authUser.js:31-39`) shows `httpOnly` cookies were considered (XSS-resistant but need CSRF protection). This is the localStorage-vs-cookie tradeoff, and the app picked the more XSS-exposed option.

## Pre-merge auth checklist (use on any PR here)

- [ ] Does this add a route? Is it under an authenticated mount, or intentionally public?
- [ ] Does it read a record by id? In a multi-user future, would it need an ownership filter?
- [ ] Does it trust any client field for identity/money? (Should be server-set.)
- [ ] Does it spread `req.body` into a write? (Mass-assignment risk — `createCRUDController/update.js:9`.)
- [ ] Does a query touch a soft-deletable model without `removed:false`?
- [ ] Does it build a query/regex from raw user input?

## Drill

Trace a request to `GET /api/invoice/read/:id` and mark every point where authorization *could* be enforced but isn't. Then write the shared-middleware ownership filter that would make the whole API multi-tenant-safe in one place, and explain why putting it per-controller is the wrong call.

**Interview angle:** Q4, Q13 ([08/03](../08-interview-prep/03-api-and-data-modeling-questions.md)), Q4/Q5 ([08/01](../08-interview-prep/01-js-ts-node-deep-dive.md)).
