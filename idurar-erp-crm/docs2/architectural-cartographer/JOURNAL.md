# Journal

## First-Pass Mental Model

This repository is a two-app JavaScript system: a Vite/React/Redux frontend in `frontend/` and an Express/Mongoose backend in `backend/`. The frontend mounts `RootApp` from `frontend/src/main.jsx`, wraps routing in React Router and Redux, then chooses either auth screens or the ERP app based on Redux auth state (`frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:1-21`, `frontend/src/apps/IdurarOs.jsx:23-72`). The backend starts from `backend/src/server.js`, connects to MongoDB, loads every model file with `globSync`, then mounts Express routers from `backend/src/app.js` (`backend/src/server.js:13-39`, `backend/src/app.js:37-43`).

## Key Discoveries

The strongest stable business flow is invoice creation and payment recording. It runs from invoice routes in `frontend/src/router/routes.jsx`, through invoice page config, ERP table/create modules, Redux ERP actions, the generic request client, protected Express routes, custom invoice/payment controllers, and Mongoose models (`frontend/src/router/routes.jsx:52-70`, `frontend/src/pages/Invoice/index.jsx:9-104`, `frontend/src/modules/ErpPanelModule/CreateItem.jsx:40-139`, `frontend/src/redux/erp/actions.js:36-118`, `frontend/src/request/request.js:27-302`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/controllers/appControllers/invoiceController/create.js:9-74`, `backend/src/controllers/appControllers/paymentController/create.js:9-86`, `backend/src/models/appModels/Invoice.js:3-179`, `backend/src/models/appModels/Payment.js:3-57`).

The repo is not a TypeScript codebase even though the prompt asks for TypeScript orientation. There are no `.ts` or `.tsx` files, while `jsconfig.json` files only configure the `@` alias (`frontend/jsconfig.json:1-9`, `backend/jsconfig.json:1-9`). The closest "type" surfaces are action type constants, Joi schemas, Mongoose schemas, and Redux state shapes (`frontend/src/context/erp/types.jsx:1-6`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`, `backend/src/models/appModels/Invoice.js:3-179`, `frontend/src/redux/erp/reducer.js:3-68`).

Several routes or setup paths reference code that is not present. Frontend routes render `Quote`, `QuoteCreate`, `QuoteRead`, `QuoteUpdate`, `PaymentMode`, and `Taxes`, but those identifiers are not imported in `frontend/src/router/routes.jsx` (`frontend/src/router/routes.jsx:1-26`, `frontend/src/router/routes.jsx:72-115`). Backend setup requires `PaymentMode` and `Taxes` models, but `backend/src/models/appModels/` contains only `Client.js`, `Invoice.js`, and `Payment.js` (`backend/src/setup/setup.js:54-67`, `backend/src/models/appModels/Client.js:1-35`, `backend/src/models/appModels/Invoice.js:1-179`, `backend/src/models/appModels/Payment.js:1-57`).

## Why These Files Were Chosen

`frontend/src/RootApp.jsx`, `frontend/src/apps/IdurarOs.jsx`, and `frontend/src/apps/ErpApp.jsx` show the frontend boot sequence, auth gate, settings preload, layout, and router placement (`frontend/src/RootApp.jsx:11-21`, `frontend/src/apps/IdurarOs.jsx:64-72`, `frontend/src/apps/ErpApp.jsx:25-90`).

`frontend/src/redux/erp/actions.js`, `frontend/src/redux/erp/reducer.js`, and `frontend/src/request/request.js` show how UI events become backend calls and how responses become table/form state (`frontend/src/redux/erp/actions.js:36-118`, `frontend/src/redux/erp/reducer.js:70-127`, `frontend/src/request/request.js:16-302`).

`backend/src/app.js`, `backend/src/routes/appRoutes/appApi.js`, `backend/src/models/utils/index.js`, and `backend/src/controllers/appControllers/index.js` show route protection and the dynamic model-to-controller convention (`backend/src/app.js:39-43`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/models/utils/index.js:4-49`, `backend/src/controllers/appControllers/index.js:1-38`).

## Where A Junior Might Get Confused

A junior may expect each route to have a hand-written route file, but app entities are discovered by model filename and wired dynamically (`backend/src/models/utils/index.js:22-47`, `backend/src/routes/appRoutes/appApi.js:28-31`). They may also assume all sidebar links are safe because they render in navigation, but some lead to missing route components (`frontend/src/apps/Navigation/NavigationContainer.jsx:68-88`, `frontend/src/router/routes.jsx:72-115`).

## Where A Mid-Level Engineer Should Slow Down

Slow down where the same concept appears in more than one state layer. CRUD state exists in Redux slices and UI context providers, while ERP modules use both Redux for server data and context for modal/panel UI state (`frontend/src/redux/rootReducer.js:11-17`, `frontend/src/layout/ErpLayout/index.jsx:8-23`, `frontend/src/context/erp/actions.jsx:3-51`).

Also slow down at query construction. The generic paginated list blocks object-style operator injection for `equal`, but the invoice-specific paginated list still places `[filter]: equal` directly into the Mongo query (`backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:18-29`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`).

## Where A Senior Engineer Should Be Skeptical

The backend `productionErrors` handler still returns the raw error object to clients, which weakens the "No stacktraces are leaked" comment above it (`backend/src/handlers/errorHandlers.js:66-77`). Authentication stores JWTs in localStorage-backed Redux persistence and sends them in the `Authorization` header, while old httpOnly cookie code is commented out (`frontend/src/redux/store.js:10-24`, `frontend/src/request/request.js:16-25`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:31-40`).

Setup deserves skepticism because it requires missing models for payment modes and taxes, which can block a clean first run (`backend/src/setup/setup.js:54-67`, `backend/src/models/utils/index.js:4-49`).

## How To Use Checkpoints

Treat each checkpoint as an engineering rehearsal. Say your answer out loud, name the exact files and line ranges, then compare it to the self-grade rubric. Strong answers connect the invoice domain to the architecture: a customer owes money, an invoice records the obligation, a payment reduces the balance, and the UI must show the new payment status.

