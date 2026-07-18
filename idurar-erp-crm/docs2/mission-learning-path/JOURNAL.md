# Mission Journal

## Mission Design Rationale

The missions start with boot flow because every unfamiliar codebase first asks, "How does this thing wake up?" The frontend boot path is small and readable, while the backend boot path introduces env vars, MongoDB, model registration, and Express routing (`frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:11-21`, `backend/src/server.js:13-39`, `backend/src/app.js:37-43`).

The middle missions use invoices because invoices connect the actual product domain to the most important technical patterns: route config, modules, Redux, Axios, auth middleware, controllers, Mongoose schemas, and money calculations (`frontend/src/pages/Invoice/index.jsx:9-104`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:41-215`, `frontend/src/redux/erp/actions.js:36-118`, `frontend/src/request/request.js:27-302`, `backend/src/controllers/appControllers/invoiceController/create.js:9-74`, `backend/src/models/appModels/Invoice.js:3-179`).

The senior missions focus on risk: missing route targets, missing setup models, raw production errors, localStorage token exposure, query construction, pagination limits, and lack of test scripts (`frontend/src/router/routes.jsx:72-115`, `backend/src/setup/setup.js:54-67`, `backend/src/handlers/errorHandlers.js:66-77`, `frontend/src/request/request.js:16-25`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`, `frontend/package.json:30-35`, `backend/package.json:8-15`).

## Why Missions Are Ordered This Way

Tier 1 builds orientation: boot, folders, contracts, components, routes, props, data, navigation. Tier 2 builds tracing skill: state, hooks, side effects, API contracts, middleware, full-stack flow, diffs, composition, runtime contracts. Tier 3 builds ownership: architecture decisions, bug prediction, performance, security, tests, and history.

## Code Paths Chosen

Frontend anchors: `main.jsx`, `RootApp.jsx`, `IdurarOs.jsx`, `ErpApp.jsx`, `routes.jsx`, `NavigationContainer.jsx`, invoice pages/modules/forms, ERP table/create modules, Redux ERP/auth/settings, and request/error handlers (`frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:1-21`, `frontend/src/apps/IdurarOs.jsx:23-72`, `frontend/src/apps/ErpApp.jsx:25-90`, `frontend/src/router/routes.jsx:28-129`, `frontend/src/apps/Navigation/NavigationContainer.jsx:51-99`, `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:22-286`).

Backend anchors: `server.js`, `app.js`, core/app routes, auth middleware, CRUD controller factory, invoice/payment controllers, app models, setup script, and error handlers (`backend/src/server.js:13-39`, `backend/src/app.js:37-52`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:5-69`, `backend/src/controllers/middlewaresControllers/createCRUDController/index.js:15-33`, `backend/src/controllers/appControllers/paymentController/create.js:9-86`).

## Skill Each Tier Develops

Junior: location memory, naming patterns, basic component reading, backend route reading, and domain vocabulary.

Mid-level: end-to-end tracing, lifecycle reasoning, API contract reading, state ownership, and diff review.

Senior: risk prediction, security/performance reasoning, test design, architecture critique, and ownership planning.

## What A Senior Engineer Would Do Differently

A senior uses checkpoints to ask, "What can break if this changes?" They do not stop at "what does this function do?" For example, when seeing invoice totals in both frontend and backend, they ask which one is authoritative and whether tests protect the backend calculation (`frontend/src/modules/ErpPanelModule/CreateItem.jsx:54-76`, `backend/src/controllers/appControllers/invoiceController/create.js:29-48`).

