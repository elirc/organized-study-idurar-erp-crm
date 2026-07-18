# Reading Map

## Mental Model

IDURAR is an invoice-centered ERP/CRM: React pages configure entity screens, reusable ERP modules turn configs into tables/forms, Redux thunks call a generic Axios request layer, Express protects `/api` routes with admin JWT middleware, dynamic route wiring maps Mongoose model files to CRUD controllers, and custom invoice/payment controllers enforce the money rules (`frontend/src/pages/Invoice/index.jsx:9-104`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:41-215`, `frontend/src/redux/erp/actions.js:36-118`, `frontend/src/request/request.js:27-302`, `backend/src/app.js:39-43`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/controllers/appControllers/invoiceController/create.js:9-74`, `backend/src/controllers/appControllers/paymentController/create.js:9-86`).

## Top 10 Files To Read In Order

1. `frontend/src/main.jsx`  
   Why now: it proves the frontend starts with one React root. Understand DOM mounting first. Explain after reading: how `RootApp` becomes the whole app (`frontend/src/main.jsx:1-6`).

2. `frontend/src/RootApp.jsx`  
   Why now: it shows React Router, Redux, Suspense, and lazy loading at the app boundary. Understand providers before pages. Explain after reading: why every page can use router and Redux state (`frontend/src/RootApp.jsx:3-21`).

3. `frontend/src/apps/IdurarOs.jsx`  
   Why now: it is the auth gate. Understand logged-in vs logged-out rendering. Explain after reading: what `isLoggedIn` decides (`frontend/src/apps/IdurarOs.jsx:23-72`).

4. `frontend/src/apps/ErpApp.jsx`  
   Why now: it loads settings before showing navigation and routes. Understand that settings are treated like app boot data. Explain after reading: why the user sees a loader until settings succeed (`frontend/src/apps/ErpApp.jsx:25-90`).

5. `frontend/src/router/routes.jsx`  
   Why now: it lists the app's public screens. Understand that route config is data. Explain after reading: which paths are real invoice/customer/payment paths and which are currently broken references (`frontend/src/router/routes.jsx:28-129`).

6. `frontend/src/modules/ErpPanelModule/DataTable.jsx`  
   Why now: it is the reusable table shell for invoice/payment-like entities. Understand table actions before forms. Explain after reading: how row actions navigate to read/update/pay/download/delete (`frontend/src/modules/ErpPanelModule/DataTable.jsx:41-215`).

7. `frontend/src/redux/erp/actions.js`  
   Why now: it converts UI intent into HTTP requests and Redux state changes. Understand the action lifecycle. Explain after reading: what `list`, `create`, `read`, and `recordPayment` do (`frontend/src/redux/erp/actions.js:36-118`, `frontend/src/redux/erp/actions.js:127-183`).

8. `backend/src/app.js`  
   Why now: it shows middleware and route mounting. Understand `/api` protection. Explain after reading: which routes are public and which require `adminAuth.isValidAuthToken` (`backend/src/app.js:21-43`).

9. `backend/src/routes/appRoutes/appApi.js`  
   Why now: it reveals dynamic CRUD routing. Understand entity route conventions. Explain after reading: how `/invoice/create` or `/payment/list` gets registered (`backend/src/routes/appRoutes/appApi.js:8-31`).

10. `backend/src/controllers/appControllers/invoiceController/create.js`  
    Why now: it is the core domain behavior. Understand validation, totals, status, persistence, and invoice numbering. Explain after reading: how line items become invoice totals and why settings are updated (`backend/src/controllers/appControllers/invoiceController/create.js:9-74`).

## Three Most Important Data Flows

1. Login flow: login form calls `auth.service.login`, backend validates admin credentials, backend returns a JWT, Redux stores auth state, and `IdurarOs` switches into the ERP app (`frontend/src/auth/auth.service.js:7-27`, `backend/src/routes/coreRoutes/coreAuth.js:8-13`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/login.js:7-57`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:14-53`, `frontend/src/redux/auth/reducer.js:21-37`, `frontend/src/apps/IdurarOs.jsx:64-72`).

2. Invoice list flow: `/invoice` renders invoice columns, `ErpPanel` resets ERP state, the data table dispatches `erp.list`, the request layer calls `GET invoice/list`, Express protects the route, the invoice list controller queries MongoDB, and Redux feeds Ant Design Table (`frontend/src/router/routes.jsx:52-55`, `frontend/src/pages/Invoice/index.jsx:21-104`, `frontend/src/modules/ErpPanelModule/index.jsx:12-35`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:151-215`, `frontend/src/redux/erp/actions.js:36-68`, `frontend/src/request/request.js:155-174`, `backend/src/app.js:39-43`, `backend/src/routes/appRoutes/appApi.js:14-17`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-67`).

3. Record payment flow: invoice row action navigates to `/invoice/pay/:id`, the module loads the invoice if needed, the payment form dispatches `erp.recordPayment`, the backend creates a payment and increments invoice credit/status, and the frontend resets state and returns to the invoice list (`frontend/src/modules/ErpPanelModule/DataTable.jsx:102-105`, `frontend/src/modules/InvoiceModule/RecordPaymentModule/index.jsx:11-36`, `frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:15-73`, `frontend/src/redux/erp/actions.js:98-118`, `backend/src/controllers/appControllers/paymentController/create.js:9-86`).

## Pre-Reading Checklist

1. Can I identify the frontend entry point and the backend entry point?
2. Can I explain where logged-in state lives?
3. Can I name the database models that actually exist?
4. Can I name one route that works and one route that appears broken?
5. Can I explain what `entity` means in this codebase?
6. Can I trace one Redux action to one HTTP request?
7. Can I distinguish UI context state from server data state?
8. Can I explain how invoice totals are calculated on both frontend and backend?
9. Can I name the middleware that protects app API routes?
10. Can I state which files define the invoice schema and payment schema?

## Red Flags Checklist

1. Route component identifiers used without imports or files (`frontend/src/router/routes.jsx:1-26`, `frontend/src/router/routes.jsx:72-115`).
2. Setup scripts requiring missing models (`backend/src/setup/setup.js:54-67`).
3. Raw error objects returned by the production error handler (`backend/src/handlers/errorHandlers.js:66-77`).
4. JWT persisted in localStorage-backed auth state (`frontend/src/redux/store.js:10-24`, `frontend/src/request/request.js:16-25`).
5. Dynamic Mongo query keys from request input (`backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`).
6. Duplicate or stale imports in router/app files (`frontend/src/router/AppRouter.jsx:1-8`, `frontend/src/apps/ErpApp.jsx:1-23`).
7. Settings must load before the ERP layout renders, so settings API failure blocks the app shell (`frontend/src/apps/ErpApp.jsx:36-90`).
8. Commented cookie auth suggests an unfinished auth hardening path (`backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:31-40`).
9. No `.ts` or `.tsx` files despite TypeScript-oriented package metadata (`frontend/package.json:37-44`, `frontend/jsconfig.json:1-9`).
10. Tests are not configured in package scripts (`frontend/package.json:30-35`, `backend/package.json:8-15`).

