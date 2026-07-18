# Senior Engineer Guide

## Architectural Critique

Scalability: 3/5. The dynamic CRUD pattern reduces boilerplate for simple entities, but missing model/controller references and route identifiers show that conventions are not enforced by tests or compile-time checks (`backend/src/models/utils/index.js:22-47`, `backend/src/routes/appRoutes/appApi.js:8-31`, `frontend/src/router/routes.jsx:72-115`).

TypeScript discipline: 1/5. The codebase has no `.ts` or `.tsx` source, and the `jsconfig` files only provide alias paths (`frontend/jsconfig.json:1-9`, `backend/jsconfig.json:1-9`). Runtime contracts exist but are scattered across Mongoose, Joi, Redux, and UI configs (`backend/src/models/appModels/Invoice.js:3-179`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`, `frontend/src/redux/erp/reducer.js:3-68`).

Separation of concerns: 3/5. Reusable ERP modules are a good boundary, but form subtotal logic, backend invoice total logic, and payment max logic duplicate financial rules across frontend and backend (`frontend/src/modules/ErpPanelModule/CreateItem.jsx:54-76`, `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:54-58`, `backend/src/controllers/appControllers/invoiceController/create.js:29-48`, `backend/src/controllers/appControllers/paymentController/create.js:30-38`).

Testability: 1/5. No frontend or backend package script runs tests, and no obvious test framework appears in dependencies (`frontend/package.json:30-45`, `backend/package.json:8-45`).

Maintainability: 3/5. The folder structure is understandable, but dynamic discovery and undefined route identifiers can create runtime failures that static checks would catch (`frontend/src/router/routes.jsx:1-26`, `frontend/src/router/routes.jsx:72-115`, `backend/src/setup/setup.js:54-67`).

Security posture: 2/5. Protected APIs use JWT auth middleware, but tokens are stored in localStorage-backed auth state and raw error objects can be returned in production (`backend/src/app.js:39-43`, `frontend/src/redux/store.js:10-24`, `frontend/src/request/request.js:16-25`, `backend/src/handlers/errorHandlers.js:66-77`).

Performance: 3/5. React lazy loading and backend compression help, but frontend effects create abort controllers without wiring them into Axios, and settings block the entire shell (`frontend/src/RootApp.jsx:3-21`, `backend/src/app.js:21-32`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:162-168`, `frontend/src/apps/ErpApp.jsx:51-90`).

## Performance Audit

Finding: abort controllers are created but not connected to requests. In `DataTable`, cleanup aborts a controller, but `erp.list` and `request.list` never receive `controller.signal` (`frontend/src/modules/ErpPanelModule/DataTable.jsx:162-168`, `frontend/src/redux/erp/actions.js:36-68`, `frontend/src/request/request.js:155-174`).

Corrected pattern:

```jsx
// Suggested replacement shape for frontend/src/modules/ErpPanelModule/DataTable.jsx:162-168
useEffect(() => {
  const controller = new AbortController();
  dispatch(erp.list({ entity, options: { page: 1, items: 10 }, signal: controller.signal }));
  return () => controller.abort();
}, [dispatch, entity]);
```

```js
// Suggested request-layer shape near frontend/src/request/request.js:155-174
list: async ({ entity, options = {}, signal }) => {
  includeToken();
  const response = await axios.get(entity + '/list' + query, { signal });
  return response.data;
}
```

Finding: backend pagination has no upper bound on `items`, so a client can request a very large page. Generic pagination parses `req.query.items` with a default of 10 but no max (`backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:1-7`).

Corrected snippet:

```js
// Suggested replacement near backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:2-4
const page = Math.max(parseInt(req.query.page, 10) || 1, 1);
const requestedLimit = parseInt(req.query.items, 10) || 10;
const limit = Math.min(Math.max(requestedLimit, 1), 100);
const skip = page * limit - limit;
```

## Security Audit

Finding: production errors return the raw `error` object despite the comment saying stack traces should not leak (`backend/src/handlers/errorHandlers.js:66-77`).

Corrected snippet:

```js
// Suggested replacement for backend/src/handlers/errorHandlers.js:71-77
exports.productionErrors = (error, req, res, next) => {
  return res.status(500).json({
    success: false,
    message: 'Internal server error',
  });
};
```

Finding: invoice-specific pagination builds a Mongo condition using a user-controlled key without the object-value guard that generic pagination has (`backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`, `backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:18-29`).

Corrected snippet:

```js
// Suggested replacement near backend/src/controllers/appControllers/invoiceController/paginatedList.js:10-28
const allowedFilters = new Set(['client', 'createdBy', 'paymentStatus', 'status']);
let filterCondition = {};
if (filter && equal !== undefined) {
  if (!allowedFilters.has(filter) || typeof equal === 'object') {
    return res.status(400).json({ success: false, result: [], message: 'Invalid filter' });
  }
  filterCondition = { [filter]: equal };
}
```

Finding: tokens are pulled from persisted auth state and attached to Axios defaults. LocalStorage exposure increases blast radius if XSS occurs (`frontend/src/redux/store.js:10-24`, `frontend/src/request/request.js:16-25`). The backend contains commented httpOnly cookie code, which suggests a known but unfinished safer direction (`backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:31-40`).

## TypeScript Discipline Review

The current contract enforcement is runtime-only. Highest-impact migration path:

1. Add TS for pure value modules first, such as `calculate`, request response shapes, and Redux state types (`frontend/src/utils/calculate.js:1-18`, `frontend/src/request/request.js:27-302`, `frontend/src/redux/erp/reducer.js:3-68`).
2. Create shared frontend types for `Invoice`, `Payment`, `Client`, and paginated responses based on Mongoose schemas (`backend/src/models/appModels/Invoice.js:3-179`, `backend/src/models/appModels/Payment.js:3-57`, `backend/src/models/appModels/Client.js:3-35`).
3. Mirror Joi validation with explicit backend input types when the backend is migrated (`backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`).

## Custom Abstractions Inventory

Dynamic CRUD controller factory: `createCRUDController(modelName)` creates create/read/update/delete/list/listAll/search/filter/summary functions from a Mongoose model (`backend/src/controllers/middlewaresControllers/createCRUDController/index.js:15-33`).

Dynamic app route factory: `routerApp(entity, controller)` registers standard routes for every app entity (`backend/src/routes/appRoutes/appApi.js:8-31`).

ERP table/form shell: `ErpPanel`, `DataTable`, `CreateItem`, and related files make invoices/payments look consistent (`frontend/src/modules/ErpPanelModule/index.jsx:12-35`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:41-215`, `frontend/src/modules/ErpPanelModule/CreateItem.jsx:40-139`).

Request facade: `request.js` centralizes token inclusion, URL construction, success notifications, and error handling (`frontend/src/request/request.js:16-302`).

Settings hooks: `useMoney` and `useDate` are consumed by table/form components to keep display formatting centralized (`frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:13-20`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:18-24`).

## Testing Assessment And One Complete Runnable Test

No test framework is configured in package scripts (`frontend/package.json:30-35`, `backend/package.json:8-15`). The riskiest untested behavior is payment creation updating invoice credit and payment status (`backend/src/controllers/appControllers/paymentController/create.js:59-77`).

Recommended test file if adding Jest or Vitest plus an in-memory MongoDB test harness:

```js
// backend/src/controllers/appControllers/paymentController/create.test.js
const paymentCreate = require('./create');

test('records a partial payment and updates invoice credit/status', async () => {
  const req = {
    admin: { _id: adminId },
    body: { invoice: invoiceId, client: clientId, amount: 25, number: 1, currency: 'USD' },
  };
  const res = mockExpressRes();

  await paymentCreate(req, res);

  const updatedInvoice = await mongoose.model('Invoice').findById(invoiceId);
  expect(updatedInvoice.credit).toBe(25);
  expect(updatedInvoice.paymentStatus).toBe('partially');
  expect(res.status).toHaveBeenCalledWith(200);
});
```

This is not currently runnable until a test runner, database test setup, and helpers like `mockExpressRes` are added.

## Bug Injection Exercise

1. Symptom: opening `/quote` crashes with a reference error. Test scenario: render `AppRouter` and navigate to `/quote`. Expected diagnosis: `Quote` is used but not imported or defined (`frontend/src/router/routes.jsx:72-87`).
2. Symptom: fresh setup fails before default payment mode creation. Test scenario: run `npm run setup` on a clean clone. Expected diagnosis: missing `PaymentMode` and `Taxes` model files (`backend/src/setup/setup.js:54-67`).
3. Symptom: requesting a huge page makes invoice list slow. Test scenario: call `/api/invoice/list?items=100000`. Expected diagnosis: no pagination limit cap (`backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-10`).
4. Symptom: a production API error exposes implementation details. Test scenario: force an exception after `productionErrors`. Expected diagnosis: raw error is returned (`backend/src/handlers/errorHandlers.js:71-76`).
5. Symptom: creating an invoice with no line items can succeed with total zero if frontend permits it. Test scenario: post valid invoice fields with `items: []`. Expected diagnosis: Joi requires array but not `.min(1)` (`backend/src/controllers/appControllers/invoiceController/schemaValidate.js:10-23`).

## Git History Learning Exercise

1. `feat(invoice): add custom total calculation before persistence`  
   Implies generic CRUD was insufficient for financial documents because totals need backend authority (`backend/src/controllers/appControllers/invoiceController/create.js:29-48`).

2. `refactor(api): generate entity routes from app model files`  
   Implies the team preferred convention over repeated route declarations (`backend/src/models/utils/index.js:22-47`, `backend/src/routes/appRoutes/appApi.js:28-31`).

3. `fix(auth): require JWT middleware for app and settings routes`  
   Implies earlier routes may have been exposed or auth boundaries were clarified (`backend/src/app.js:39-43`).

4. `feat(payment): update invoice credit when recording payment`  
   Implies payments are not standalone records; they mutate invoice balance (`backend/src/controllers/appControllers/paymentController/create.js:66-77`).

5. `chore(frontend): centralize server API configuration`  
   Implies environment-specific API URLs had been duplicated before `serverApiConfig.js` (`frontend/src/config/serverApiConfig.js:1-19`).

## If I Owned This Codebase

1. Restore or remove broken quote/tax/payment-mode paths. Effort: medium. Impact: high. Broken route and setup references block trust in basic navigation/setup (`frontend/src/router/routes.jsx:72-115`, `backend/src/setup/setup.js:54-67`).
2. Add backend tests around invoice create/update/payment create. Effort: medium. Impact: high. These are money rules (`backend/src/controllers/appControllers/invoiceController/create.js:29-48`, `backend/src/controllers/appControllers/paymentController/create.js:59-77`).
3. Harden production error responses and invoice filters. Effort: small. Impact: high (`backend/src/handlers/errorHandlers.js:71-76`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`).
4. Add pagination limits everywhere. Effort: small. Impact: medium (`backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:1-7`).
5. Introduce TypeScript or at least JSDoc schema typedefs for invoices/payments. Effort: large. Impact: medium-high (`backend/src/models/appModels/Invoice.js:3-179`, `frontend/src/redux/erp/reducer.js:3-68`).
6. Move auth toward httpOnly secure cookies or a clearly threat-modeled token storage decision. Effort: large. Impact: high (`frontend/src/request/request.js:16-25`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:31-40`).

