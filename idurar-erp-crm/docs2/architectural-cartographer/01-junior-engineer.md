# Junior Engineer Guide

## Local Setup

The repo ships separate frontend and backend package files, both targeting Node `20.9.0` and npm `10.2.4` (`frontend/package.json:4-7`, `backend/package.json:4-7`). The backend exposes `npm run setup`, `npm run dev`, `npm run start`, and reset/upgrade scripts (`backend/package.json:8-15`). The frontend exposes `npm run dev`, `npm run build`, `npm run lint`, `npm run preview`, and `npm run dev:remote` (`frontend/package.json:30-35`).

Reconstructed setup:

```bash
cd backend
npm install
# create backend/.env with DATABASE="your-mongodb-uri" and JWT_SECRET="your-secret"
npm run setup
npm run dev

cd ../frontend
npm install
npm run dev
```

The official install instructions mention MongoDB setup, `backend/.env`, `DATABASE`, backend install/setup/dev, frontend install/dev, and localhost:3000 (`INSTALLATION-INSTRUCTIONS.md:13-91`). The frontend hardcodes local API/download URLs to `http://localhost:8888/` when not in production or remote mode (`frontend/src/config/serverApiConfig.js:1-16`). The backend listens on `process.env.PORT || 8888` (`backend/src/server.js:34-39`).

Setup gap: `npm run setup` requires `PaymentMode` and `Taxes` models, but the inspected app model folder only includes `Client`, `Invoice`, and `Payment` models (`backend/src/setup/setup.js:54-67`, `backend/src/models/appModels/Client.js:1-35`, `backend/src/models/appModels/Invoice.js:1-179`, `backend/src/models/appModels/Payment.js:1-57`). Expect setup to fail until those missing models are restored or the script is fixed.

## Folder Orientation

Frontend:

`frontend/src/main.jsx` mounts React, `frontend/src/RootApp.jsx` installs global providers, `frontend/src/apps/` contains the auth gate and ERP app shell, `frontend/src/router/` maps paths to pages, `frontend/src/pages/` provides route-level screen configs, `frontend/src/modules/` contains reusable feature modules, `frontend/src/redux/` owns global server/auth/settings state, `frontend/src/context/` owns local UI panel/modal context, `frontend/src/request/` wraps Axios, and `frontend/src/components/` holds reusable UI pieces (`frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:1-21`, `frontend/src/apps/IdurarOs.jsx:23-72`, `frontend/src/router/routes.jsx:28-129`, `frontend/src/redux/rootReducer.js:11-17`, `frontend/src/request/request.js:27-302`).

Backend:

`backend/src/server.js` loads env vars, connects MongoDB, loads models, and starts Express; `backend/src/app.js` builds the middleware stack and mounts routers; `backend/src/routes/` maps HTTP paths to controllers; `backend/src/controllers/` holds auth, CRUD, app-specific, PDF, and setup logic; `backend/src/models/` defines Mongoose models and model discovery; `backend/src/middlewares/` holds settings, upload, and inventory helpers (`backend/src/server.js:13-39`, `backend/src/app.js:1-52`, `backend/src/routes/appRoutes/appApi.js:1-33`, `backend/src/controllers/appControllers/index.js:1-38`, `backend/src/models/utils/index.js:1-49`).

Most important frontend folders:

`router`: route config is centralized in `routes.jsx`, and `AppRouter` flattens route groups into `useRoutes` (`frontend/src/router/routes.jsx:28-129`, `frontend/src/router/AppRouter.jsx:15-43`).

`modules`: feature modules are reusable shells. `ErpPanelModule` provides table/read/create/update/delete style behaviors for invoice/payment-like entities (`frontend/src/modules/ErpPanelModule/index.jsx:12-35`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:41-215`, `frontend/src/modules/ErpPanelModule/CreateItem.jsx:40-139`).

`redux`: Redux contains auth, CRUD, ERP, advanced CRUD, and settings reducers (`frontend/src/redux/rootReducer.js:1-19`).

Most important backend folders:

`routes`: `coreAuth` exposes public auth routes; `coreApi` exposes protected admin/settings routes; `appApi` dynamically exposes app entity CRUD routes (`backend/src/routes/coreRoutes/coreAuth.js:1-15`, `backend/src/routes/coreRoutes/coreApi.js:1-52`, `backend/src/routes/appRoutes/appApi.js:1-33`).

`controllers`: generic CRUD lives under `middlewaresControllers/createCRUDController`, while custom invoice/payment logic lives under `appControllers` (`backend/src/controllers/middlewaresControllers/createCRUDController/index.js:1-35`, `backend/src/controllers/appControllers/invoiceController/index.js:1-20`, `backend/src/controllers/appControllers/paymentController/create.js:1-86`).

`models`: Mongoose app models define the database contracts, and `models/utils` discovers app model files to create entity route metadata (`backend/src/models/appModels/Invoice.js:3-179`, `backend/src/models/appModels/Payment.js:3-57`, `backend/src/models/utils/index.js:4-49`).

## Frontend Entry Point Walkthrough

```jsx
// frontend/src/main.jsx:1-6
import { createRoot } from 'react-dom/client'; // React 18 root API.
import RootApp from './RootApp';               // The app boundary with providers.

const root = createRoot(document.getElementById('root')); // Bind to index.html #root.
root.render(<RootApp />);                                // Everything else flows from RootApp.
```

```jsx
// frontend/src/RootApp.jsx:11-21
export default function RoutApp() {
  return (
    <BrowserRouter>                 // Enables URL-based routing.
      <Provider store={store}>      // Makes Redux state/actions available to descendants.
        <Suspense fallback={<PageLoader />}> // Handles lazy-loaded app modules.
          <IdurarOs />              // Chooses auth screens or ERP app.
        </Suspense>
      </Provider>
    </BrowserRouter>
  );
}
```

## Backend Entry Point Walkthrough

```js
// backend/src/server.js:13-39
require('dotenv').config({ path: '.env' });       // Load backend/.env first.
require('dotenv').config({ path: '.env.local' }); // Allow local override.

mongoose.connect(process.env.DATABASE);           // MongoDB is required before serving.

const modelsFiles = globSync('./src/models/**/*.js');
for (const filePath of modelsFiles) {
  require(path.resolve(filePath));                // Register every Mongoose model.
}

const app = require('./app');                     // Build Express app after models exist.
app.set('port', process.env.PORT || 8888);
const server = app.listen(app.get('port'), () => {
  console.log(`Express running -> On PORT : ${server.address().port}`);
});
```

```js
// backend/src/app.js:37-43
app.use('/api', coreAuthRouter);                         // Login/reset/logout auth routes.
app.use('/api', adminAuth.isValidAuthToken, coreApiRouter); // Protected admin/settings API.
app.use('/api', adminAuth.isValidAuthToken, erpApiRouter);  // Protected app entity API.
app.use('/download', coreDownloadRouter);                // PDF download path.
app.use('/public', corePublicRouter);                    // Public uploads/static path.
```

## TypeScript Orientation

This repo has no `.ts` or `.tsx` source files. Treat it as JavaScript with contracts expressed in other ways: Mongoose schemas define persistence shape, Joi schemas validate request payloads, Redux reducers define state shape, and constants define action names (`backend/src/models/appModels/Invoice.js:3-179`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`, `frontend/src/redux/erp/reducer.js:3-68`, `frontend/src/context/erp/types.jsx:1-6`). If TypeScript existed, these same contracts would become interfaces such as `Invoice`, `InvoiceItem`, `Payment`, and `ErpListState`.

## React Component Anatomy 1: Simple Status Tag

```jsx
// frontend/src/components/Tag/index.jsx:4-19
export function StatusTag({ status = 'draft' }) { // Prop contract: accepts a status string.
  const translate = useLanguage();                // Domain text goes through localization.
  let color = () => {                             // UI color is derived from status.
    return status === 'draft'
      ? 'cyan'
      : status === 'sent'
      ? 'blue'
      : status === 'accepted'
      ? 'green'
      : status === 'expired'
      ? 'orange'
      : 'red';
  };

  return <Tag color={color()}>{translate(status)}</Tag>; // Render translated label.
}
```

## React Component Anatomy 2: Invoice Page Config

```jsx
// frontend/src/pages/Invoice/index.jsx:9-21
export default function Invoice() {
  const translate = useLanguage();      // Labels come from the localization layer.
  const { dateFormat } = useDate();     // Formatting is centralized in settings hooks.
  const entity = 'invoice';             // This string drives backend URL construction.
  const { moneyFormatter } = useMoney();

  const searchConfig = {
    entity: 'client',                   // Invoice list filters by client.
    displayLabels: ['name'],
    searchFields: 'name',
  };
  const deleteModalLabels = ['number', 'client.name'];
```

```jsx
// frontend/src/pages/Invoice/index.jsx:93-104
const configPage = {
  entity,              // Passed into modules so they can call /invoice routes.
  ...Labels,           // Human-facing labels.
};
const config = {
  ...configPage,
  dataTableColumns,    // Table knows how to display invoice fields.
  searchConfig,        // DataTable knows how to filter by client.
  deleteModalLabels,
};

return <InvoiceDataTableModule config={config} />; // Page delegates behavior to module.
```

## React Component Anatomy 3: Complex ERP Data Table

```jsx
// frontend/src/modules/ErpPanelModule/DataTable.jsx:41-52
export default function DataTable({ config, extra = [] }) {
  const translate = useLanguage();
  let { entity, dataTableColumns, disableAdd = false, searchConfig } = config;
  const { result: listResult, isLoading: listIsLoading } = useSelector(selectListItems);
  const { pagination, items: dataSource } = listResult; // Redux result feeds the Ant table.
  const { erpContextAction } = useErpContext();
  const { modal } = erpContextAction; // Context owns local UI modal state.
```

```jsx
// frontend/src/modules/ErpPanelModule/DataTable.jsx:102-105
const handleRecordPayment = (record) => {
  dispatch(erp.currentItem({ data: record })); // Preserve clicked invoice in Redux.
  navigate(`/invoice/pay/${record._id}`);      // Route to payment workflow.
};
```

```jsx
// frontend/src/modules/ErpPanelModule/DataTable.jsx:153-172
const handelDataTableLoad = (pagination) => {
  const options = { page: pagination.current || 1, items: pagination.pageSize || 10 };
  dispatch(erp.list({ entity, options })); // Server-backed pagination.
};

const dispatcher = () => {
  dispatch(erp.list({ entity }));          // Initial table load.
};

useEffect(() => {
  const controller = new AbortController();
  dispatcher();
  return () => {
    controller.abort();                    // Intended cleanup, but request is not wired to signal.
  };
}, []);
```

## Backend Anatomy 1: Simple Generic Create

```js
// backend/src/controllers/middlewaresControllers/createCRUDController/create.js:1-14
const create = async (Model, req, res) => {
  req.body.removed = false;        // Soft-delete flag is normalized at creation.
  const result = await new Model({
    ...req.body,                   // Request body becomes document fields.
  }).save();

  return res.status(200).json({
    success: true,
    result,
    message: 'Successfully Created the document in Model ',
  });
};
```

## Backend Anatomy 2: Dynamic App Routes

```js
// backend/src/routes/appRoutes/appApi.js:8-26
const routerApp = (entity, controller) => {
  router.route(`/${entity}/create`).post(catchErrors(controller['create']));
  router.route(`/${entity}/read/:id`).get(catchErrors(controller['read']));
  router.route(`/${entity}/update/:id`).patch(catchErrors(controller['update']));
  router.route(`/${entity}/delete/:id`).delete(catchErrors(controller['delete']));
  router.route(`/${entity}/search`).get(catchErrors(controller['search']));
  router.route(`/${entity}/list`).get(catchErrors(controller['list']));
  router.route(`/${entity}/listAll`).get(catchErrors(controller['listAll']));
  router.route(`/${entity}/filter`).get(catchErrors(controller['filter']));
  router.route(`/${entity}/summary`).get(catchErrors(controller['summary']));

  if (entity === 'invoice' || entity === 'quote' || entity === 'payment') {
    router.route(`/${entity}/mail`).post(catchErrors(controller['mail']));
  }
};
```

## Backend Anatomy 3: Invoice Create

```js
// backend/src/controllers/appControllers/invoiceController/create.js:12-23
const { error, value } = schema.validate(body); // Validate request before money math.
if (error) {
  const { details } = error;
  return res.status(400).json({
    success: false,
    result: null,
    message: details[0]?.message,
  });
}

const { items = [], taxRate = 0, discount = 0 } = value;
```

```js
// backend/src/controllers/appControllers/invoiceController/create.js:29-48
items.map((item) => {
  let total = calculate.multiply(item['quantity'], item['price']); // Precise currency math.
  subTotal = calculate.add(subTotal, total);                       // Sum all invoice lines.
  item['total'] = total;                                           // Persist computed line total.
});
taxTotal = calculate.multiply(subTotal, taxRate / 100);
total = calculate.add(subTotal, taxTotal);

body['subTotal'] = subTotal;
body['taxTotal'] = taxTotal;
body['total'] = total;
body['items'] = items;

let paymentStatus = calculate.sub(total, discount) === 0 ? 'paid' : 'unpaid';
body['paymentStatus'] = paymentStatus;
body['createdBy'] = req.admin._id; // Auth middleware attached req.admin.
```

## Domain Glossary

Admin: the authenticated owner user. The only allowed role enum is `owner` (`backend/src/models/coreModels/Admin.js:30-34`).

Client/customer: the CRM contact billed by invoices and payments (`backend/src/models/appModels/Client.js:13-22`).

Invoice: the core sales document. It belongs to an admin, references a client, has line items, tax, total, credit, payment status, PDF filename, and status (`backend/src/models/appModels/Invoice.js:9-179`).

Item: one invoice line with name, description, quantity, price, and computed total (`backend/src/models/appModels/Invoice.js:51-95`).

Payment: money received against an invoice. It references client and invoice, records amount, currency, ref, description, and creator (`backend/src/models/appModels/Payment.js:9-55`).

Credit: total paid amount accumulated on an invoice (`backend/src/models/appModels/Invoice.js:118-121`, `backend/src/controllers/appControllers/paymentController/create.js:66-77`).

Payment status: invoice state derived from total, discount, credit, and payment amount: `unpaid`, `partially`, or `paid` (`backend/src/models/appModels/Invoice.js:132-136`, `backend/src/controllers/appControllers/paymentController/create.js:59-72`).

Setting: a typed key/value stored in MongoDB and loaded at app startup for finance, company, money format, and other behavior (`backend/src/models/coreModels/Setting.js:13-37`, `frontend/src/apps/ErpApp.jsx:36-43`).

Entity: a lowercase string such as `invoice`, `payment`, or `client` used to construct frontend request URLs and backend dynamic routes (`frontend/src/pages/Invoice/index.jsx:12-19`, `backend/src/models/utils/index.js:34-46`).

## Junior Socratic Checkpoint

1. What file proves the frontend starts with `RootApp`?
2. What line decides whether the user sees auth screens or the ERP app?
3. What exact string makes invoice requests go to `/invoice/...`?
4. What middleware protects `/api/invoice/list`?
5. Where is invoice total calculated on the backend?
6. Where is invoice line total calculated on the frontend?
7. Which missing models can block backend setup?

How to self-grade:

A strong answer cites `frontend/src/main.jsx:1-6`, `frontend/src/apps/IdurarOs.jsx:64-72`, `frontend/src/pages/Invoice/index.jsx:12-19`, `backend/src/app.js:39-43`, `backend/src/controllers/appControllers/invoiceController/create.js:29-48`, `frontend/src/modules/ErpPanelModule/CreateItem.jsx:89-103`, and `backend/src/setup/setup.js:54-67`. It should explain the invoice like a bill at a service desk: the UI drafts it, the backend verifies the math, MongoDB stores it, and payment later reduces the balance.

