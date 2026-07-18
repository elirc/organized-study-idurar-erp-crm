# Reference Artifacts

### Mission 25: Write the Docs That Don't Exist
**Tier:** Senior  
**Time Estimate:** 90 minutes  
**Goal:** Turn codebase discoveries into operational references.
**The Concept:** Documentation is not a brochure. It is the checklist the next engineer uses while holding a real ticket.
**Design Intent Before You Read the Code:** Useful docs should start from workflow: setup, navigate, review, debug, change, own, explain.
**Find It In The Code:** Use the same anchors as this mission suite: boot, routes, Redux, request layer, auth middleware, invoice/payment controllers, models, setup script, and known gaps (`frontend/src/main.jsx:1-6`, `backend/src/server.js:13-39`, `frontend/src/redux/erp/actions.js:36-118`, `backend/src/controllers/appControllers/paymentController/create.js:9-86`, `backend/src/setup/setup.js:54-67`).
**The Aha Moment:** The best docs teach the next action, not just the current structure.
**Socratic Checkpoint:**  
1. What workflow does each doc support?  
2. What code path anchors each claim?  
3. What known gaps must not be hidden?  
4. What would a new engineer do after reading?  
5. How would you verify the docs stay current?  
How to self-grade: Strong answers identify code anchors, workflow outcomes, and maintenance triggers.
**Connects To:** Future feature stories because documentation should support real implementation work.

## Doc 1: Junior Onboarding Checklist

- Confirm Node/npm versions from both package files (`frontend/package.json:4-7`, `backend/package.json:4-7`).
- Create `backend/.env` with `DATABASE` and `JWT_SECRET`; backend code reads them during boot/auth (`backend/src/server.js:13-19`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:15-20`).
- Run backend install/setup/dev and frontend install/dev, but expect setup risk from missing `PaymentMode` and `Taxes` models (`INSTALLATION-INSTRUCTIONS.md:35-91`, `backend/src/setup/setup.js:54-67`).
- First PR checklist: identify route, page config, module, Redux action, request method, backend controller, schema, and model.

## Doc 2: Architecture Guide For New Engineers

Navigate from outside in: boot files, app shell, route config, page config, reusable module, Redux action, request facade, Express route, controller, model (`frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:11-21`, `frontend/src/router/routes.jsx:28-129`, `frontend/src/pages/Invoice/index.jsx:9-104`, `frontend/src/redux/erp/actions.js:36-118`, `frontend/src/request/request.js:27-302`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/models/appModels/Invoice.js:3-179`).

## Doc 3: Code Review Checklist

- Does navigation match actual routes and imports? (`frontend/src/apps/Navigation/NavigationContainer.jsx:51-99`, `frontend/src/router/routes.jsx:1-129`)
- Does form payload match Joi validation and Mongoose schema? (`frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:66-286`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`, `backend/src/models/appModels/Invoice.js:3-179`)
- Does server data update Redux success/failure buckets? (`frontend/src/redux/erp/actions.js:36-118`, `frontend/src/redux/erp/reducer.js:70-127`)
- Does backend protect the route and sanitize query inputs? (`backend/src/app.js:39-43`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`)

## Doc 4: Debugging Playbook

Blank app after login: inspect settings load in `ErpApp` (`frontend/src/apps/ErpApp.jsx:36-90`).

Invoice list not updating: trace `DataTable` dispatch, `erp.list`, `request.list`, backend route, and invoice list controller (`frontend/src/modules/ErpPanelModule/DataTable.jsx:153-172`, `frontend/src/redux/erp/actions.js:36-68`, `frontend/src/request/request.js:155-174`, `backend/src/routes/appRoutes/appApi.js:14-17`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-67`).

Login/auth failure: inspect login service, auth route, login controller, `authUser`, and auth reducer (`frontend/src/auth/auth.service.js:7-27`, `backend/src/routes/coreRoutes/coreAuth.js:8-13`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/login.js:7-57`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:14-53`, `frontend/src/redux/auth/reducer.js:21-37`).

## Doc 5: Change Playbook

1. Create branch.
2. Find the route/page/module stack.
3. Modify frontend UI and state.
4. Modify backend route/controller/schema/model if persistence changes.
5. Manually test login, settings boot, list, create/read/update, and any payment effect.
6. Review for route gaps, setup gaps, query risks, and raw error leaks.
7. Merge only after documenting user-visible behavior and residual risk.

## Doc 6: Senior Ownership Notes

Monitor setup reliability, invoice/payment correctness, auth token handling, query safety, and missing tests. Do not over-refactor reusable ERP modules until the broken quote/tax/payment-mode references are resolved, because refactoring on top of missing pieces can confuse cause and effect (`frontend/src/router/routes.jsx:72-115`, `backend/src/setup/setup.js:54-67`).

## Doc 7: Interview Walkthrough

Practice answer: "I worked with a React/Redux frontend and Express/Mongoose backend for an ERP/CRM. The most important flow is invoice/payment. A page config defines `entity = 'invoice'`, reusable modules render tables and forms, Redux thunks call an Axios request layer, Express protects app API routes with JWT middleware, dynamic routes map entity names to controllers, and custom invoice/payment controllers enforce totals and payment status. The biggest risks I found were missing route/model targets, runtime-only contracts, production error leakage, localStorage token exposure, and untested financial flows." Cite `frontend/src/pages/Invoice/index.jsx:9-104`, `frontend/src/redux/erp/actions.js:36-118`, `backend/src/app.js:39-43`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/controllers/appControllers/invoiceController/create.js:9-74`, `backend/src/controllers/appControllers/paymentController/create.js:9-86`.

