# Tier 1 Junior Missions

### Mission 1: The App's Heartbeat
**Tier:** Junior  
**Time Estimate:** 25 minutes  
**Goal:** Trace how the frontend and backend start.
**The Concept:** Every app has a heartbeat: the first file that makes the system alive. In this ERP, that is like opening the office: frontend unlocks the counter, backend connects to the filing cabinet.
**Design Intent Before You Read the Code:** Frontend boot owns rendering and providers; backend boot owns env vars, MongoDB, model registration, and server start. Bad boot code makes every feature look broken.
**Find It In The Code:** Open `frontend/src/main.jsx:1-6`, `frontend/src/RootApp.jsx:11-21`, `backend/src/server.js:13-39`, `backend/src/app.js:37-43`.

```jsx
// frontend/src/main.jsx:1-6
import { createRoot } from 'react-dom/client'; // React creates the browser root.
import RootApp from './RootApp';               // RootApp owns global providers.
const root = createRoot(document.getElementById('root')); // Attach to index.html.
root.render(<RootApp />);                      // Start the frontend tree.
```

```js
// backend/src/server.js:13-39
require('dotenv').config({ path: '.env' });       // Load DATABASE/JWT_SECRET.
require('dotenv').config({ path: '.env.local' }); // Optional local override.
mongoose.connect(process.env.DATABASE);           // Connect before serving.
for (const filePath of modelsFiles) require(path.resolve(filePath)); // Register models.
const app = require('./app');                     // Build Express app.
app.set('port', process.env.PORT || 8888);        // Default backend port.
app.listen(app.get('port'), () => {});            // Begin accepting requests.
```

**The Aha Moment:** Before features exist, providers, database connection, and router mounting must exist.
**Socratic Checkpoint:**  
1. What file starts React?  
2. What file starts Express?  
3. Why are models loaded before `app`?  
4. What env var is required for MongoDB?  
5. What port does the backend use by default?  
How to self-grade: Strong answers cite `frontend/src/main.jsx:1-6`, `backend/src/server.js:13-39`, and explain that Mongoose models must be registered before controllers call `mongoose.model(...)`.
**Connects To:** Mission 2 because folders make sense after you know the boot files; Mission 5 because backend routes start from `app.js`.

### Mission 2: The Folder Mental Map
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Build a folder map you can use without guessing.
**The Concept:** Folders are departments in the ERP office: pages greet users, modules do workflows, redux tracks shared state, backend controllers make decisions, models store records.
**Design Intent Before You Read the Code:** Frontend folders separate route screens, reusable modules, state, requests, and components. Backend folders separate boot, routes, controllers, models, and middleware.
**Find It In The Code:** Open `frontend/src/router/routes.jsx:28-129`, `frontend/src/redux/rootReducer.js:11-17`, `backend/src/app.js:37-52`, `backend/src/controllers/appControllers/index.js:1-38`, `backend/src/models/utils/index.js:4-49`.

```js
// frontend/src/redux/rootReducer.js:11-17
const rootReducer = combineReducers({
  auth: authReducer,             // login/session state.
  crud: crudReducer,             // generic CRUD state.
  erp: erpReducer,               // invoice/payment ERP state.
  adavancedCrud: adavancedCrudReducer, // advanced CRUD area.
  settings: settingsReducer,     // global app settings.
});
```

```js
// backend/src/controllers/appControllers/index.js:29-33
routesList.forEach(({ modelName, controllerName }) => {
  if (!hasCustomControllers.includes(controllerName)) {
    controllers[controllerName] = createCRUDController(modelName); // Fill gaps with generic CRUD.
  }
});
```

**The Aha Moment:** The repo is organized around reusable entity workflows, not one custom screen per feature.
**Socratic Checkpoint:**  
1. Which frontend folder owns routes?  
2. Which frontend folder owns server state?  
3. Which backend folder owns Mongoose schemas?  
4. Which file combines reducers?  
5. How are missing custom controllers handled?  
How to self-grade: Strong answers cite the exact files above and explain dynamic CRUD fallback.
**Connects To:** Mission 4 because components live inside this map; Mission 13 because middleware/route folders form the backend chain.

### Mission 3: TypeScript Is a Contract
**Tier:** Junior  
**Time Estimate:** 20 minutes  
**Goal:** Learn where contracts live when TypeScript is absent.
**The Concept:** TypeScript is like a preflight checklist for invoices. This repo lacks that checklist, so runtime contracts do the checking at the counter.
**Design Intent Before You Read the Code:** Since no `.ts` or `.tsx` files exist, look for contracts in schemas, reducers, constants, and request validation.
**Find It In The Code:** Open `frontend/jsconfig.json:1-9`, `backend/jsconfig.json:1-9`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`, `frontend/src/redux/erp/reducer.js:3-68`.

```js
// backend/src/controllers/appControllers/invoiceController/schemaValidate.js:2-24
const schema = Joi.object({
  client: Joi.alternatives().try(Joi.string(), Joi.object()).required(), // Client must exist.
  number: Joi.number().required(),                                      // Invoice number.
  items: Joi.array().items(Joi.object({                                 // Runtime item shape.
    itemName: Joi.string().required(),
    quantity: Joi.number().required(),
    price: Joi.number().required(),
    total: Joi.number().required(),
  }).required()).required(),
  taxRate: Joi.alternatives().try(Joi.number(), Joi.string()).required(),
});
```

**The Aha Moment:** In this repo, "type safety" is mostly runtime validation and convention.
**Socratic Checkpoint:**  
1. Are there TypeScript source files?  
2. What does `jsconfig` provide?  
3. What validates invoice input?  
4. What defines invoice persistence shape?  
5. What defines ERP state shape?  
How to self-grade: Strong answers cite `frontend/jsconfig.json:1-9`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`, `backend/src/models/appModels/Invoice.js:3-179`, and `frontend/src/redux/erp/reducer.js:3-68`.
**Connects To:** Mission 17 because runtime contracts do hidden type work; Mission 23 because missing tests matter more without TS.

### Mission 4: Your First React Component
**Tier:** Junior  
**Time Estimate:** 25 minutes  
**Goal:** Read a small component from props to render.
**The Concept:** A component is a clerk at the ERP counter: it receives a small request, consults settings or translation, and returns a UI response.
**Design Intent Before You Read the Code:** A simple component should have clear inputs, little state, and predictable output.
**Find It In The Code:** Open `frontend/src/components/Tag/index.jsx:1-19`.

```jsx
// frontend/src/components/Tag/index.jsx:4-19
export function StatusTag({ status = 'draft' }) { // Input prop with default.
  const translate = useLanguage();                // All labels go through localization.
  let color = () => {                             // Status maps to visual color.
    return status === 'draft' ? 'cyan' : status === 'sent' ? 'blue' : 'red';
  };
  return <Tag color={color()}>{translate(status)}</Tag>; // Render the translated status.
}
```

**The Aha Moment:** A small component is easiest to understand by asking "input, derived value, output?"
**Socratic Checkpoint:**  
1. What prop enters this component?  
2. What default value is used?  
3. What hook is called?  
4. What library component is rendered?  
5. What happens for an unknown status?  
How to self-grade: Strong answers cite `frontend/src/components/Tag/index.jsx:4-19` and mention fallback red.
**Connects To:** Mission 6 because props become contracts; Mission 16 because larger modules are composed from small components.

### Mission 5: Your First Node.js Route
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Trace one route from Express mount to controller.
**The Concept:** A backend route is a service window. The URL decides which window opens, middleware checks the badge, and the controller performs the work.
**Design Intent Before You Read the Code:** Auth routes are public; app routes are protected; app entity routes are generated.
**Find It In The Code:** Open `backend/src/app.js:37-43`, `backend/src/routes/coreRoutes/coreAuth.js:8-13`, `backend/src/routes/appRoutes/appApi.js:8-31`.

```js
// backend/src/app.js:39-41
app.use('/api', coreAuthRouter);                            // Login is public.
app.use('/api', adminAuth.isValidAuthToken, coreApiRouter); // Settings/admin protected.
app.use('/api', adminAuth.isValidAuthToken, erpApiRouter);  // Invoice/payment/client protected.
```

```js
// backend/src/routes/appRoutes/appApi.js:8-17
router.route(`/${entity}/create`).post(catchErrors(controller['create'])); // POST /api/invoice/create.
router.route(`/${entity}/read/:id`).get(catchErrors(controller['read']));  // GET one record.
router.route(`/${entity}/list`).get(catchErrors(controller['list']));      // GET paginated list.
```

**The Aha Moment:** `/api/invoice/list` exists because `invoice` is an entity discovered from a model file.
**Socratic Checkpoint:**  
1. Which routes are public?  
2. Which middleware protects app routes?  
3. What route handles invoice creation?  
4. What wrapper catches controller errors?  
5. Where does `entity` come from?  
How to self-grade: Strong answers cite `backend/src/app.js:39-43`, `backend/src/routes/appRoutes/appApi.js:8-31`, and `backend/src/models/utils/index.js:22-47`.
**Connects To:** Mission 12 because routes define the API contract; Mission 13 because middleware controls access.

### Mission 6: Props Are a Typed Contract
**Tier:** Junior  
**Time Estimate:** 25 minutes  
**Goal:** See how props carry feature configuration.
**The Concept:** Props are a work order. For an invoice table, `config` tells the reusable module which entity, labels, columns, and search behavior to use.
**Design Intent Before You Read the Code:** Page components should configure; reusable modules should execute behavior.
**Find It In The Code:** Open `frontend/src/pages/Invoice/index.jsx:93-104`, `frontend/src/modules/InvoiceModule/InvoiceDataTableModule/index.jsx:6-22`, `frontend/src/modules/ErpPanelModule/index.jsx:12-35`.

```jsx
// frontend/src/pages/Invoice/index.jsx:93-104
const config = {
  ...configPage,       // entity and labels.
  dataTableColumns,    // visual table contract.
  searchConfig,        // filter/search contract.
  deleteModalLabels,   // delete confirmation contract.
};
return <InvoiceDataTableModule config={config} />; // Pass work order down.
```

**The Aha Moment:** `config` lets one table module serve many ERP entities.
**Socratic Checkpoint:**  
1. Which prop carries invoice table configuration?  
2. What does `entity` control?  
3. Where is the extra "Record Payment" action added?  
4. Which module resets ERP state?  
5. Why is this better than copying a table for every entity?  
How to self-grade: Strong answers cite all three files and connect `entity` to request URLs.
**Connects To:** Mission 14 because config begins the full-stack trace; Mission 16 because this is composition.

### Mission 7: Following Data Into the App
**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** Trace invoice list data from backend response into a table.
**The Concept:** Data travels like an invoice moving from filing cabinet to front desk: backend returns records, Redux stores them, and the table displays them.
**Design Intent Before You Read the Code:** Data fetching should have loading, success, failure, and render phases.
**Find It In The Code:** Open `frontend/src/modules/ErpPanelModule/DataTable.jsx:151-215`, `frontend/src/redux/erp/actions.js:36-68`, `frontend/src/request/request.js:155-174`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-67`.

```jsx
// frontend/src/modules/ErpPanelModule/DataTable.jsx:203-211
<Table
  columns={dataTableColumns} // Page-configured columns.
  rowKey={(item) => item._id} // MongoDB document id.
  dataSource={dataSource}    // Redux list.result.items.
  pagination={pagination}    // Redux pagination from backend.
  loading={listIsLoading}    // Redux loading state.
  onChange={handelDataTableLoad} // Server pagination callback.
/>
```

**The Aha Moment:** The table is not "fetching"; it is displaying Redux state that a thunk fetched.
**Socratic Checkpoint:**  
1. Where is `dataSource` read from?  
2. What thunk loads list data?  
3. What request method performs HTTP?  
4. What backend controller returns pagination?  
5. What UI prop shows loading?  
How to self-grade: Strong answers cite `frontend/src/modules/ErpPanelModule/DataTable.jsx:47-49`, `frontend/src/redux/erp/actions.js:36-68`, `frontend/src/request/request.js:155-174`, and `backend/src/controllers/appControllers/invoiceController/paginatedList.js:49-63`.
**Connects To:** Mission 9 because state needs a home; Mission 14 because this is one segment of the full flow.

### Mission 8: Navigation Is the App's Skeleton
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Understand visible navigation and route configuration.
**The Concept:** Navigation is the office hallway. Routes are the rooms. A hallway sign pointing to a missing room is a real product bug.
**Design Intent Before You Read the Code:** Navigation links should match actual routes and route components.
**Find It In The Code:** Open `frontend/src/apps/Navigation/NavigationContainer.jsx:51-99`, `frontend/src/router/routes.jsx:28-129`.

```jsx
// frontend/src/apps/Navigation/NavigationContainer.jsx:63-88
{ key: 'invoice', label: <Link to={'/invoice'}>{translate('invoices')}</Link> } // Real route.
{ key: 'quote', label: <Link to={'/quote'}>{translate('quote')}</Link> }        // Route exists but component is missing.
{ key: 'taxes', label: <Link to={'/taxes'}>{translate('taxes')}</Link> }        // Route references undefined Taxes.
```

```jsx
// frontend/src/router/routes.jsx:72-115
{ path: '/quote', element: <Quote /> }      // Quote is not imported above.
{ path: '/payment/mode', element: <PaymentMode /> } // PaymentMode is not imported.
{ path: '/taxes', element: <Taxes /> }      // Taxes is not imported.
```

**The Aha Moment:** Navigation can look complete while routes are runtime-broken.
**Socratic Checkpoint:**  
1. Which navigation links point to invoice/payment/customer?  
2. Which route entries reference undefined identifiers?  
3. Why can this slip past without TypeScript?  
4. What user-visible symptom would appear?  
5. What files would you fix first?  
How to self-grade: Strong answers cite `frontend/src/apps/Navigation/NavigationContainer.jsx:63-88`, `frontend/src/router/routes.jsx:72-115`, and mention missing imports/files.
**Connects To:** Mission 15 because diff review should catch route/navigation mismatches; Mission 19 because this is bug prediction.

