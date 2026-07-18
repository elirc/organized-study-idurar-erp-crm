# Mid-Level Engineer Guide

## Architecture Diagram

```text
Browser
  |
  | React root
  v
frontend/src/main.jsx -> RootApp -> IdurarOs
  |                         |
  |                         +-- not logged in -> AuthRouter
  |                         +-- logged in -> ErpApp
  |
  v
ErpApp loads settings -> Navigation + AppRouter
  |
  v
Page config, for example Invoice
  |
  v
Reusable module, for example ErpPanel/DataTable/CreateItem
  |
  v
Redux ERP thunk -> request.js -> Axios -> /api/{entity}/{action}
  |
  v
Express app -> auth middleware -> appApi dynamic router
  |
  v
Custom controller or generic CRUD controller
  |
  v
Mongoose model -> MongoDB -> JSON response -> Redux -> React table/form
```

This diagram is grounded in the actual boot and route chain: React mounts `RootApp`, `RootApp` installs Router/Redux/Suspense, `IdurarOs` branches by `isLoggedIn`, `ErpApp` loads settings and renders `AppRouter`, Redux actions call the request layer, Express protects app APIs with `adminAuth.isValidAuthToken`, and app routes are generated from model metadata (`frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:11-21`, `frontend/src/apps/IdurarOs.jsx:23-72`, `frontend/src/apps/ErpApp.jsx:36-90`, `frontend/src/redux/erp/actions.js:36-118`, `frontend/src/request/request.js:27-302`, `backend/src/app.js:39-43`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/models/utils/index.js:22-47`).

## Type System Deep Dive

There is no TypeScript source. The system relies on runtime contracts:

Mongoose persistence contract: invoice fields, enums, references, default values, and autopopulate are in the schema (`backend/src/models/appModels/Invoice.js:3-179`).

Joi request contract: invoice create/update requires client, number, year, status, dates, items, quantity, price, total, and tax rate (`backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`).

Redux state contract: `erpReducer` predefines buckets for `current`, `list`, `create`, `update`, `delete`, `read`, `recordPayment`, `search`, `summary`, and `mail` (`frontend/src/redux/erp/reducer.js:3-68`).

Action string contract: UI context action names are string constants, not TypeScript discriminated unions (`frontend/src/context/erp/types.jsx:1-6`).

If this were TypeScript, the highest-value first types would be:

```ts
// Conceptual only; no .ts files exist in this repo.
type PaymentStatus = 'unpaid' | 'paid' | 'partially'; // Mirrors backend enum.
type InvoiceStatus = 'draft' | 'pending' | 'sent' | 'refunded' | 'cancelled' | 'on hold';
interface InvoiceItem { itemName: string; description?: string; quantity: number; price: number; total: number; }
interface Invoice { _id: string; client: Client; items: InvoiceItem[]; total: number; credit: number; paymentStatus: PaymentStatus; }
```

The reason these contracts matter is practical: invoice totals and payment status are financial correctness rules, not cosmetic UI behavior (`backend/src/controllers/appControllers/invoiceController/create.js:29-48`, `backend/src/controllers/appControllers/paymentController/create.js:59-77`).

## State Management Deep Dive

The invoice list state lives in Redux under `erp.list`. Its initial shape includes `items`, `pagination`, `isLoading`, and `isSuccess` (`frontend/src/redux/erp/reducer.js:7-19`). The data table reads `selectListItems`, extracts `pagination` and `items`, and passes them to Ant Design `Table` (`frontend/src/modules/ErpPanelModule/DataTable.jsx:47-49`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:204-211`).

The lifecycle is:

```js
// frontend/src/redux/erp/actions.js:36-68
list: ({ entity, options = { page: 1, items: 10 } }) => async (dispatch) => {
  dispatch({ type: REQUEST_LOADING, keyState: 'list', payload: null }); // Table spinner.
  let data = await request.list({ entity, options });                   // HTTP read.
  if (data.success === true) {
    const result = {
      items: data.result,
      pagination: {
        current: parseInt(data.pagination.page, 10),
        pageSize: options?.items || 10,
        total: parseInt(data.pagination.count, 10),
      },
    };
    dispatch({ type: REQUEST_SUCCESS, keyState: 'list', payload: result }); // Table data.
  } else {
    dispatch({ type: REQUEST_FAILED, keyState: 'list', payload: null });
  }
}
```

UI state has a different home. `ErpContextProvider` wraps ERP pages, and context actions open/close modal/panels (`frontend/src/layout/ErpLayout/index.jsx:8-23`, `frontend/src/context/erp/actions.jsx:3-51`). Mid-level rule: server state goes in Redux, local workflow chrome goes in context.

## API Contract Map

Auth:

`POST /api/login` is public and maps to `adminAuth.login`; `POST /api/logout` is protected by `isValidAuthToken` (`backend/src/routes/coreRoutes/coreAuth.js:8-13`).

Core protected:

Admin profile and settings routes are protected when mounted in `app.js` and include `/admin/read/:id`, `/admin/profile/update`, `/setting/list`, `/setting/updateBySettingKey/:settingKey?`, and `/setting/upload/:settingKey?` (`backend/src/app.js:39-43`, `backend/src/routes/coreRoutes/coreApi.js:14-51`).

App protected:

Every discovered app entity gets create, read, update, delete, search, list, listAll, filter, and summary routes (`backend/src/routes/appRoutes/appApi.js:8-18`). Invoice/payment/quote also get mail routes, and quote gets convert (`backend/src/routes/appRoutes/appApi.js:19-25`). The quote branches are currently unsafe because quote model/controller files are not present in inspected model files (`backend/src/models/utils/index.js:4-49`).

Frontend request conventions:

`request.create` calls `POST {entity}/create`, `request.read` calls `GET {entity}/read/{id}`, `request.update` calls `PATCH {entity}/update/{id}`, `request.delete` calls `DELETE {entity}/delete/{id}`, and `request.list` calls `GET {entity}/list?...` after attaching the auth token (`frontend/src/request/request.js:16-25`, `frontend/src/request/request.js:27-83`, `frontend/src/request/request.js:102-174`).

## Component Interaction Map

`Invoice` page builds `config`, `InvoiceDataTableModule` wraps `ErpPanel`, `ErpPanel` resets ERP state and renders `DataTable` plus `Delete`, `DataTable` dispatches `erp.list`, then renders columns and row actions (`frontend/src/pages/Invoice/index.jsx:9-104`, `frontend/src/modules/InvoiceModule/InvoiceDataTableModule/index.jsx:6-22`, `frontend/src/modules/ErpPanelModule/index.jsx:12-35`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:41-215`).

`InvoiceCreate` builds config, `CreateInvoiceModule` passes `InvoiceForm` to `CreateItem`, `CreateItem` owns form submission and subtotal computation, and `InvoiceForm` owns invoice-specific fields (`frontend/src/pages/Invoice/InvoiceCreate.jsx:4-20`, `frontend/src/modules/InvoiceModule/CreateInvoiceModule/index.jsx:5-11`, `frontend/src/modules/ErpPanelModule/CreateItem.jsx:40-139`, `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:22-286`).

## Full-Stack Feature Trace: Record A Payment

1. User opens the invoice table. `Invoice` configures `entity = 'invoice'` and passes invoice columns to the module (`frontend/src/pages/Invoice/index.jsx:12-21`, `frontend/src/pages/Invoice/index.jsx:93-104`).
2. Table row action adds a `recordPayment` menu item from `InvoiceDataTableModule` (`frontend/src/modules/InvoiceModule/InvoiceDataTableModule/index.jsx:6-21`).
3. `DataTable` handles `recordPayment`, dispatches the current invoice, and navigates to `/invoice/pay/:id` (`frontend/src/modules/ErpPanelModule/DataTable.jsx:102-105`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:117-136`).
4. `RecordPaymentModule` reads the route id, uses the list cache if possible, otherwise dispatches `erp.read`, then places the invoice into the `recordPayment` action bucket (`frontend/src/modules/InvoiceModule/RecordPaymentModule/index.jsx:11-36`).
5. `RecordPayment` computes `maxAmount` as `total - discount - credit`, submits the form, and dispatches `erp.recordPayment({ entity: 'payment' })` (`frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:26-58`).
6. The thunk calls `request.create({ entity: 'payment' })`, so Axios sends `POST /api/payment/create` with auth (`frontend/src/redux/erp/actions.js:98-118`, `frontend/src/request/request.js:27-40`).
7. Express mounts app API behind `adminAuth.isValidAuthToken`, and `appApi` registers `/payment/create` dynamically (`backend/src/app.js:39-43`, `backend/src/routes/appRoutes/appApi.js:8-31`).
8. `paymentController.create` rejects zero amount, loads the invoice, computes max amount, creates payment, creates a PDF filename, pushes payment id to invoice, increments credit, and sets payment status (`backend/src/controllers/appControllers/paymentController/create.js:9-86`).
9. The thunk stores the payment result and sets the current item to `data.result.invoice`; the UI resets record-payment state, reloads invoice list, and navigates back to `/invoice/` (`frontend/src/redux/erp/actions.js:107-118`, `frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:33-40`).

## Diff Reading Exercise

Hypothetical change: "Add a new invoice status called `approved`."

Read the diff like this:

1. Check backend enum first. The invoice schema currently allows `draft`, `pending`, `sent`, `refunded`, `cancelled`, and `on hold` (`backend/src/models/appModels/Invoice.js:148-152`).
2. Check frontend form options. The invoice form currently offers `draft`, `pending`, and `sent` (`frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:118-136`).
3. Check display mappings. The simple `StatusTag` knows `accepted` and `expired`, which do not match the invoice enum, and falls back to red for unknown statuses (`frontend/src/components/Tag/index.jsx:4-19`).
4. Check validation. Joi requires status as a string but does not restrict allowed values (`backend/src/controllers/appControllers/invoiceController/schemaValidate.js:6-7`).
5. Strong review comment: "This change must update the Mongoose enum, form options, display colors, translations, and probably Joi validation, otherwise a status can render inconsistently or fail only at persistence time."

## Non-Obvious Architectural Patterns

Dynamic CRUD by model filename: app model filenames become route metadata and controller names (`backend/src/models/utils/index.js:22-47`). This reduces repeated route code but makes missing/renamed files dangerous.

Custom controller override: `appControllers/index.js` loads custom controller directories first, then fills missing controllers with generic CRUD (`backend/src/controllers/appControllers/index.js:12-33`). This lets invoices override create/update/list while clients can stay generic.

Entity-string API client: frontend request helpers construct URLs from the `entity` string, so page config controls backend endpoint choice (`frontend/src/pages/Invoice/index.jsx:12-19`, `frontend/src/request/request.js:27-302`).

Runtime settings as boot dependency: `ErpApp` dispatches settings list in `useLayoutEffect` and refuses to render the shell until settings succeed (`frontend/src/apps/ErpApp.jsx:36-90`). This is useful because money/date/localization settings are globally needed, but fragile if settings API fails.

## Mid-Level Socratic Checkpoint

1. Why is `entity` more than just a label?
2. What code path turns a model file into an API route?
3. Why does invoice creation need a custom controller instead of generic CRUD?
4. Where does server state live versus modal/panel UI state?
5. What happens if settings fail to load during app boot?
6. Why is `invoiceController/paginatedList.js` riskier than generic `paginatedList.js`?
7. How does record-payment state return the user to the invoice list?
8. What files must change to add a new invoice status safely?

How to self-grade:

A strong answer names `frontend/src/pages/Invoice/index.jsx:12-19`, `backend/src/models/utils/index.js:22-47`, `backend/src/routes/appRoutes/appApi.js:28-31`, `backend/src/controllers/appControllers/invoiceController/create.js:29-48`, `frontend/src/redux/erp/reducer.js:3-68`, `frontend/src/context/erp/actions.jsx:3-51`, `frontend/src/apps/ErpApp.jsx:51-90`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`, and `frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:33-40`.

