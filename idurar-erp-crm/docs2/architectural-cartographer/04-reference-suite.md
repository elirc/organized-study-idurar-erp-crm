# Reference Suite

## Doc 1: Junior Onboarding Guide

Start with the boot chain: `frontend/src/main.jsx` renders `RootApp`, `RootApp` provides Router/Redux/Suspense, `IdurarOs` gates by `isLoggedIn`, and `ErpApp` loads settings before rendering navigation and routes (`frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:11-21`, `frontend/src/apps/IdurarOs.jsx:23-72`, `frontend/src/apps/ErpApp.jsx:36-90`).

Then read the backend boot chain: `server.js` loads env, connects MongoDB, registers models, imports `app`, and starts port 8888 by default (`backend/src/server.js:13-39`). `app.js` mounts public auth routes, protected core/app API routes, download routes, and public routes (`backend/src/app.js:37-43`).

Day-one setup uses backend and frontend separately. Backend needs `DATABASE` and `JWT_SECRET`; frontend local dev points at `http://localhost:8888/api/` (`INSTALLATION-INSTRUCTIONS.md:21-33`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:15-20`, `frontend/src/config/serverApiConfig.js:1-16`).

## Doc 2: Mid-Level Architecture Guide

The main architectural move is entity configuration. A page such as `Invoice` sets `entity = 'invoice'`, defines columns and labels, and delegates behavior to reusable modules (`frontend/src/pages/Invoice/index.jsx:9-104`). Reusable modules dispatch Redux ERP actions, which call the request facade with the same entity string (`frontend/src/modules/ErpPanelModule/DataTable.jsx:151-172`, `frontend/src/redux/erp/actions.js:36-118`, `frontend/src/request/request.js:27-302`).

The backend mirrors that convention. App model files are discovered, lowercased into entity names, converted into controller names, and registered as routes (`backend/src/models/utils/index.js:22-47`, `backend/src/routes/appRoutes/appApi.js:8-31`). Custom controllers override generic CRUD when domain rules matter, as invoices do (`backend/src/controllers/appControllers/index.js:12-33`, `backend/src/controllers/appControllers/invoiceController/index.js:1-20`).

## Doc 3: Senior Ownership Guide

Own this codebase by protecting three things: setup reliability, money correctness, and auth/security boundaries. Setup currently references missing `PaymentMode` and `Taxes` models (`backend/src/setup/setup.js:54-67`). Money correctness lives in invoice create/update and payment create (`backend/src/controllers/appControllers/invoiceController/create.js:29-48`, `backend/src/controllers/appControllers/invoiceController/update.js:45-68`, `backend/src/controllers/appControllers/paymentController/create.js:30-77`). Auth boundaries depend on JWT verification and route mounting (`backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:5-69`, `backend/src/app.js:39-43`).

## Doc 4: Code Review Guide

Review order:

1. Route reachability: check `routes.jsx`, navigation, backend route registration, and model/controller existence (`frontend/src/router/routes.jsx:28-129`, `frontend/src/apps/Navigation/NavigationContainer.jsx:51-99`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/models/utils/index.js:4-49`).
2. Data contract: check UI form fields, Joi schema, and Mongoose schema (`frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:66-286`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`, `backend/src/models/appModels/Invoice.js:3-179`).
3. State lifecycle: check thunk loading/success/failure, reducer bucket, selector, and component consumer (`frontend/src/redux/erp/actions.js:36-118`, `frontend/src/redux/erp/reducer.js:70-127`, `frontend/src/redux/erp/selectors.js:1-22`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:47-49`).
4. Security: check token inclusion, auth middleware, query construction, and error response (`frontend/src/request/request.js:16-25`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:11-58`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`, `backend/src/handlers/errorHandlers.js:71-76`).

## Doc 5: Debugging Guide

If the screen is blank after login, check whether settings loaded; `ErpApp` returns `PageLoader` until `selectSettings.isSuccess` is true (`frontend/src/apps/ErpApp.jsx:36-90`).

If an invoice table is empty, trace `DataTable -> erp.list -> request.list -> /api/invoice/list -> invoice paginatedList` (`frontend/src/modules/ErpPanelModule/DataTable.jsx:153-172`, `frontend/src/redux/erp/actions.js:36-68`, `frontend/src/request/request.js:155-174`, `backend/src/routes/appRoutes/appApi.js:14-17`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-67`).

If auth redirects unexpectedly, inspect `errorHandler` JWT logic and backend `isValidAuthToken` responses (`frontend/src/request/errorHandler.js:41-70`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:14-68`).

If setup fails, inspect missing app models required by setup (`backend/src/setup/setup.js:54-67`).

## Doc 6: Change Playbook

To add or modify a feature:

1. Locate the route and page config. Example: invoices live at `/invoice`, `/invoice/create`, `/invoice/read/:id`, `/invoice/update/:id`, and `/invoice/pay/:id` (`frontend/src/router/routes.jsx:52-70`).
2. Locate the reusable module and form/table component (`frontend/src/modules/InvoiceModule/InvoiceDataTableModule/index.jsx:6-22`, `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:22-286`).
3. Locate Redux action and request method (`frontend/src/redux/erp/actions.js:36-118`, `frontend/src/request/request.js:27-302`).
4. Locate backend route and controller (`backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/controllers/appControllers/invoiceController/index.js:1-20`).
5. Update validation and schema together (`backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`, `backend/src/models/appModels/Invoice.js:3-179`).
6. Test the user-visible flow manually until automated tests exist.

## Doc 7: Interview Walkthrough

Talk track:

"This is a split React and Express/Mongo ERP/CRM. The frontend starts in `main.jsx`, wraps the app in Router and Redux, gates by auth, then loads settings before showing the ERP shell. Feature pages are mostly configs: invoice sets an entity string, columns, and labels, then reusable ERP modules handle tables, forms, and navigation. Redux thunks use a generic request layer that constructs URLs from the entity. The backend protects `/api` routes with JWT middleware, discovers app models dynamically, generates standard CRUD routes, and lets custom controllers override behavior when business rules matter. Invoices are the best example: backend create validates the invoice, calculates totals, persists it, assigns a PDF filename, and increments invoice numbering; payments create payment records and update invoice credit/payment status." Cite `frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:11-21`, `frontend/src/apps/IdurarOs.jsx:23-72`, `frontend/src/apps/ErpApp.jsx:36-90`, `frontend/src/pages/Invoice/index.jsx:9-104`, `frontend/src/redux/erp/actions.js:36-118`, `frontend/src/request/request.js:27-302`, `backend/src/app.js:39-43`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/controllers/appControllers/invoiceController/create.js:9-74`, and `backend/src/controllers/appControllers/paymentController/create.js:9-86`.

