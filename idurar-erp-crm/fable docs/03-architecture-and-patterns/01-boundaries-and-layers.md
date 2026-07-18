# Boundaries & Layers

## The layers

IDURAR is a classic layered app, thin at every layer:

```
Route          URL → controller           routes/appRoutes/appApi.js
  │            (generated, no logic)
Controller     validation + business logic  controllers/appControllers/*
  │            (custom) OR generic CRUD      controllers/middlewaresControllers/createCRUDController/*
Model          shape + persistence           models/appModels/*
  │
MongoDB
```

On the frontend:

```
Page           config (columns, labels)      pages/*
  │
Module         orchestration + forms         modules/*
  │
Redux          server-state cache            redux/erp/*
  │
Request        network + token               request/request.js
  │
API
```

## What each layer owns — and must not own

| Layer | Owns | Must NOT own | Boundary check |
| --- | --- | --- | --- |
| Route | URL→handler, `catchErrors` wrapping | any logic | `appApi.js` has zero business logic ✓ |
| Controller | validation, money math, invariant maintenance | HTTP-agnostic reuse (it's coupled to `req`/`res`) | see leak below |
| Model | schema, refs, validators | request handling | clean ✓ |
| Page (FE) | display config | data fetching | clean ✓ |
| Redux | server-state cache, loading flags | UI-only state | mostly clean; some over-use |
| Request | axios, token, envelope handling | rendering, business rules | clean ✓ |

## Good boundaries in this repo

**Route ↔ controller is clean.** `appApi.js:8-26` contains no logic — it maps URLs to controller methods and wraps them in `catchErrors`. All logic lives one layer down. You can understand the API surface without reading any business code.

**Model ↔ everything is clean.** Schemas are pure shape (`Invoice.js`); they don't know about HTTP or Redux. You can reason about the data independently.

**The Request layer is a real chokepoint.** Every API call goes through `request/request.js`, which centralizes token injection (`:16-25`) and the success/error notification handling. Nothing bypasses it. That's a well-drawn boundary — token logic lives in exactly one place.

## Boundary leaks (where the model breaks down)

**1. Business logic is coupled to Express.** Controllers take `(req, res)` and both read input and write responses (`invoiceController/create.js` reads `req.body`, computes totals, and calls `res.status(200).json`). The money math is *not* extractable to a testable pure function without dragging `req`/`res` along. A cleaner boundary would be a service layer: `invoiceService.create(data)` returns a result, and the controller only adapts HTTP↔service. This leak is why the app has no unit-testable business logic — a real limitation for the testing module.

**2. The "generic" table knows about invoices.** `DataTable.jsx:102-105` has a `handleRecordPayment` and the action menu special-cases invoice actions — entity-specific behavior leaking into a component that claims to be generic. The boundary between "generic table" and "invoice table" is blurred.

**3. Presentation math vs authoritative math.** The frontend computes totals for display (`InvoiceForm.jsx:54-58`, `CreateItem.jsx:63-71`) and the backend recomputes them authoritatively (`invoiceController/create.js:30-43`). This is *correct* (never trust the client), but it means the same rule lives in two layers and can drift — a boundary tension rather than a clean leak.

**4. Frontend/backend duplication across the process boundary.** `helpers.js` `calculate` (BE) and `utils/calculate.js` (FE); `useMoney` exists on both sides. The process boundary forces duplication the module system can't bridge (CJS vs ESM). Not a mistake so much as an inherent cost of the split.

## The security boundary

The single most important boundary is `app.js:39-43` — the line between authenticated and unauthenticated. It's enforced *positionally* (middleware presence), not by default. Three mounts are public: the auth routes (correct), `/download` (a problem), `/public` (guarded by path-traversal check). "Secure by omission" is fragile — see [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md).

## Ownership & blast radius

When you change a layer, the blast radius tells you what to test:
- Change a **model** field → affects the controller that reads it, the frontend column that renders it, and any Joi schema. Widest radius.
- Change a **generic controller** (`createCRUDController/*`) → affects *every* entity without an override. Very wide — a bug here breaks clients, payments, and settings at once.
- Change a **page config** → affects one screen. Narrowest.

This is why good-first-tickets favor page configs and single custom controllers over the generic factory.

## Drill

Pick `invoiceController/create.js`. Draw the line between "HTTP adapter" responsibilities and "business logic" responsibilities *within the single function*. Then sketch the function signature of a `invoiceService.create(data, adminId)` that would extract the business logic, and note what becomes unit-testable once you do. Self-grade — Strong: your service function takes plain data (no `req`/`res`) and returns a result or throws, and you can name the test cases it unlocks (totals, overpay guard, status derivation).
