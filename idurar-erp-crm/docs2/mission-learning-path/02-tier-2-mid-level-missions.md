# Tier 2 Mid-Level Missions

### Mission 9: State Has a Home and a Reason
**Tier:** Mid-Level  
**Time Estimate:** 35 minutes  
**Goal:** Distinguish server data state from UI workflow state.
**The Concept:** Invoices live in the filing system; modals are just the office doors opening and closing. Store durable data and temporary UI movement in different places.
**Design Intent Before You Read the Code:** Redux owns server-backed auth/settings/ERP data; context owns modal/panel state local to a layout.
**Find It In The Code:** Open `frontend/src/redux/rootReducer.js:11-17`, `frontend/src/redux/erp/reducer.js:3-68`, `frontend/src/layout/ErpLayout/index.jsx:8-23`, `frontend/src/context/erp/actions.jsx:3-51`.

```js
// frontend/src/redux/erp/reducer.js:7-19
list: {
  result: { items: [], pagination: { current: 1, pageSize: 10, total: 1 } }, // Server list.
  isLoading: false, // Fetch lifecycle.
  isSuccess: false,
}
```

```jsx
// frontend/src/layout/ErpLayout/index.jsx:8-23
return (
  <ErpContextProvider> // Local workflow UI context for pages using ERP modules.
    <Content>{children}</Content>
  </ErpContextProvider>
);
```

**The Aha Moment:** Redux answers "what data do we have?" while context answers "which workflow panel is open?"
**Socratic Checkpoint:**  
1. What reducers exist?  
2. Where is invoice list state initialized?  
3. Where is modal state controlled?  
4. Why not put every modal in Redux?  
5. What state survives route changes more naturally?  
How to self-grade: Strong answers cite all listed files and separate server data from local UI chrome.
**Connects To:** Mission 14 because full-stack traces cross state layers; Mission 16 because composition uses both.

### Mission 10: The Custom Hook Ecosystem
**Tier:** Mid-Level  
**Time Estimate:** 30 minutes  
**Goal:** Understand custom hooks and their tradeoffs.
**The Concept:** Hooks are reusable desk tools. A good one saves repeated work; a risky one hides side effects.
**Design Intent Before You Read the Code:** Hooks should expose small, predictable state machines and clean up subscriptions.
**Find It In The Code:** Open `frontend/src/hooks/useFetch.jsx:1-32`, `frontend/src/hooks/useOnFetch.jsx:1-22`, `frontend/src/hooks/useResponsive.jsx:1-68`, `frontend/src/apps/ErpApp.jsx:32-90`.

```jsx
// frontend/src/hooks/useResponsive.jsx:40-68
export default function useResponsive() {
  if (isBrowser && !listening) {              // Global listener initialized once.
    info = {};
    calculate();
    window.addEventListener('resize', handleResize);
    listening = true;
  }
  const [state, setState] = useState(info);   // Component-local snapshot.
  useEffect(() => {
    const subscriber = () => setState(info);  // Subscribe this component.
    subscribers.add(subscriber);
    return () => { subscribers.delete(subscriber); }; // Cleanup.
  }, []);
  return { screenSize: state, isMobile: !state.md };
}
```

**The Aha Moment:** Hooks are mini state machines; read their inputs, state, effects, cleanup, and return value.
**Socratic Checkpoint:**  
1. Which hook decides mobile layout?  
2. Which hook wraps arbitrary fetch callbacks?  
3. What cleanup does `useResponsive` perform?  
4. What dependency in `useFetch` could cause repeated fetch logic confusion?  
5. Where is `useResponsive` consumed?  
How to self-grade: Strong answers cite `frontend/src/hooks/useResponsive.jsx:40-68`, `frontend/src/hooks/useFetch.jsx:9-23`, and `frontend/src/apps/ErpApp.jsx:32-90`.
**Connects To:** Mission 11 because hooks often contain side effects; Mission 21 because subscription cleanup affects performance.

### Mission 11: Side Effects Are Promises to the System
**Tier:** Mid-Level  
**Time Estimate:** 35 minutes  
**Goal:** Audit effects that fetch, reset, or navigate.
**The Concept:** A side effect is a promise: "when this state changes, I will do this outside render." In an ERP, broken promises duplicate invoices or leave stale payment status.
**Design Intent Before You Read the Code:** Effects should have correct dependencies and cleanup that actually reaches the async work.
**Find It In The Code:** Open `frontend/src/apps/ErpApp.jsx:36-38`, `frontend/src/modules/ErpPanelModule/index.jsx:21-27`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:162-168`, `frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:33-40`.

```jsx
// frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:33-40
useEffect(() => {
  if (isSuccess) {
    form.resetFields();                              // Clear successful form.
    dispatch(erp.resetAction({ actionType: 'recordPayment' })); // Reset state bucket.
    dispatch(erp.list({ entity }));                  // Refresh invoice list.
    navigate(`/${entity}/`);                         // Return to list view.
  }
}, [isSuccess]);
```

**The Aha Moment:** Effects are where state changes become network calls, resets, or navigation.
**Socratic Checkpoint:**  
1. What effect loads settings?  
2. What effect resets ERP state on panel mount?  
3. Which cleanup creates an abort controller but does not wire it to Axios?  
4. What effect navigates after payment success?  
5. What dependency arrays would you inspect in review?  
How to self-grade: Strong answers cite the four code paths and mention that `AbortController` is currently cosmetic in the table.
**Connects To:** Mission 14 because full flows are made of side effects; Mission 21 because incorrect effects can waste work.

### Mission 12: The Full API Contract
**Tier:** Mid-Level  
**Time Estimate:** 40 minutes  
**Goal:** Map frontend request helpers to backend route handlers.
**The Concept:** API contracts are the handoff slip between the counter and the filing cabinet. If the slip says `invoice/list`, the backend must have that window open.
**Design Intent Before You Read the Code:** One frontend request facade should match backend route conventions exactly.
**Find It In The Code:** Open `frontend/src/request/request.js:27-302`, `backend/src/routes/appRoutes/appApi.js:8-31`, `backend/src/routes/coreRoutes/coreApi.js:30-51`, `backend/src/routes/coreRoutes/coreAuth.js:8-13`.

```js
// frontend/src/request/request.js:27-83
create: async ({ entity, jsonData }) => axios.post(entity + '/create', jsonData), // POST /api/{entity}/create
read: async ({ entity, id }) => axios.get(entity + '/read/' + id),                // GET /api/{entity}/read/:id
update: async ({ entity, id, jsonData }) => axios.patch(entity + '/update/' + id, jsonData),
```

```js
// backend/src/routes/appRoutes/appApi.js:8-17
router.route(`/${entity}/create`).post(catchErrors(controller['create'])); // Matches request.create.
router.route(`/${entity}/read/:id`).get(catchErrors(controller['read']));  // Matches request.read.
router.route(`/${entity}/update/:id`).patch(catchErrors(controller['update']));
```

**The Aha Moment:** The string `entity` is part of the API contract, not just display config.
**Socratic Checkpoint:**  
1. What URL does `request.create({ entity: 'invoice' })` call?  
2. What backend route handles it?  
3. What methods include notifications?  
4. Where is the auth token added?  
5. Which setting routes are not generated dynamically?  
How to self-grade: Strong answers cite request methods, `includeToken`, app routes, and core setting routes.
**Connects To:** Mission 13 because middleware decides whether contracts are reachable; Mission 14 because API contracts carry the feature trace.

### Mission 13: The Middleware Chain
**Tier:** Mid-Level  
**Time Estimate:** 35 minutes  
**Goal:** Trace a protected request through Express middleware.
**The Concept:** Middleware is the badge scanner before the ERP office doors. It checks identity before invoice records are opened.
**Design Intent Before You Read the Code:** Auth middleware should extract token, verify it, load user, confirm session, and attach user to request.
**Find It In The Code:** Open `backend/src/app.js:21-43`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:5-69`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/authUser.js:14-53`.

```js
// backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:11-23
const authHeader = req.headers['authorization'];       // Read Bearer header.
const token = authHeader && authHeader.split(' ')[1];  // Extract token.
if (!token) return res.status(401).json({ jwtExpired: true }); // Reject missing token.
const verified = jwt.verify(token, process.env[jwtSecret]);    // Verify signature.
```

```js
// backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:32-58
const [user, userPassword] = await Promise.all([userPromise, userPasswordPromise]); // Load user/session.
if (!loggedSessions.includes(token)) return res.status(401).json({ jwtExpired: true }); // Token must be active.
req[reqUserName] = user; // Attach req.admin for controllers.
next();                  // Continue to route handler.
```

**The Aha Moment:** Controllers trust `req.admin` because middleware created it.
**Socratic Checkpoint:**  
1. Where is the token read from?  
2. What env var signs/verifies JWTs?  
3. What collection stores active sessions?  
4. What property is attached to `req`?  
5. Which routes bypass this middleware?  
How to self-grade: Strong answers cite `isValidAuthToken`, `authUser`, `AdminPassword`, and `coreAuth`.
**Connects To:** Mission 22 because auth storage affects security; Mission 14 because `req.admin` is used in invoice/payment writes.

### Mission 14: End-to-End Feature Trace
**Tier:** Mid-Level  
**Time Estimate:** 55 minutes  
**Goal:** Trace record payment from click to database update.
**The Concept:** Recording a payment is like marking a client's bill: the UI selects the invoice, the API records payment, and the invoice balance changes.
**Design Intent Before You Read the Code:** The flow must preserve invoice id, client id, amount limit, auth user, and resulting payment status.
**Find It In The Code:** Open `frontend/src/modules/ErpPanelModule/DataTable.jsx:102-136`, `frontend/src/modules/InvoiceModule/RecordPaymentModule/index.jsx:11-36`, `frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:15-73`, `frontend/src/redux/erp/actions.js:98-118`, `backend/src/controllers/appControllers/paymentController/create.js:9-86`.

```jsx
// frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:42-58
const onSubmit = (fieldsValue) => {
  const { _id: invoice } = currentInvoice; // Preserve invoice id.
  const client = currentInvoice.client && currentInvoice.client._id; // Preserve client id.
  fieldsValue = { ...fieldsValue, invoice, client }; // Payload for payment create.
  dispatch(erp.recordPayment({ entity: 'payment', jsonData: fieldsValue })); // POST payment/create.
};
```

```js
// backend/src/controllers/appControllers/paymentController/create.js:59-77
let paymentStatus =
  calculate.sub(total, discount) === calculate.add(credit, amount)
    ? 'paid'       // Fully paid after this payment.
    : calculate.add(credit, amount) > 0
    ? 'partially'  // Some money received.
    : 'unpaid';

await Invoice.findOneAndUpdate(
  { _id: req.body.invoice },
  { $push: { payment: paymentId.toString() }, $inc: { credit: amount }, $set: { paymentStatus } },
  { new: true, runValidators: true }
).exec();
```

**The Aha Moment:** A payment is both a new document and a mutation to the invoice's balance.
**Socratic Checkpoint:**  
1. What click starts the flow?  
2. Where does the invoice id come from?  
3. Where is max amount computed on the frontend?  
4. Where is max amount enforced on the backend?  
5. What Mongo update changes invoice credit?  
How to self-grade: Strong answers cite all listed files and explain both frontend convenience and backend authority.
**Connects To:** Mission 19 because this flow contains likely bugs; Mission 23 because it deserves a test.

### Mission 15: Read the Diff Like an Engineer
**Tier:** Mid-Level  
**Time Estimate:** 40 minutes  
**Goal:** Practice reviewing a realistic change.
**The Concept:** A diff is not a list of edits; it is a proposed change to behavior. For invoices, always ask what happens to money, status, and persistence.
**Design Intent Before You Read the Code:** Review all surfaces touched by a behavior: UI options, validation, schema enum, display, backend calculation, and tests.
**Find It In The Code:** Open `backend/src/models/appModels/Invoice.js:148-152`, `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:118-136`, `frontend/src/components/Tag/index.jsx:4-19`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:6-7`.

```js
// backend/src/models/appModels/Invoice.js:148-152
status: {
  type: String,
  enum: ['draft', 'pending', 'sent', 'refunded', 'cancelled', 'on hold'], // Persistence truth.
  default: 'draft',
},
```

**The Aha Moment:** A status change is not done until UI, validation, persistence, display, and tests agree.
**Socratic Checkpoint:**  
1. Where is invoice status persisted?  
2. Where can a user select status?  
3. Where is status color derived?  
4. Does Joi restrict allowed status values?  
5. What would your review comment say?  
How to self-grade: Strong answers cite all files and request an end-to-end status consistency check.
**Connects To:** Mission 18 because diffs reveal architecture decisions; Mission 19 because review is bug prevention.

### Mission 16: Composition Over Inheritance
**Tier:** Mid-Level  
**Time Estimate:** 35 minutes  
**Goal:** See how reusable modules build features through composition.
**The Concept:** The ERP app builds invoices like assembling a document packet: page config, layout, table, form, and row components each handle one layer.
**Design Intent Before You Read the Code:** Reuse should come from passing components/config, not subclassing behavior.
**Find It In The Code:** Open `frontend/src/modules/InvoiceModule/CreateInvoiceModule/index.jsx:1-11`, `frontend/src/modules/ErpPanelModule/CreateItem.jsx:40-139`, `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:22-286`.

```jsx
// frontend/src/modules/InvoiceModule/CreateInvoiceModule/index.jsx:5-11
export default function CreateInvoiceModule({ config }) {
  return (
    <ErpLayout>
      <CreateItem config={config} CreateForm={InvoiceForm} /> // Generic creator + invoice form.
    </ErpLayout>
  );
}
```

**The Aha Moment:** `CreateItem` is reusable because it receives the specific form as a component.
**Socratic Checkpoint:**  
1. Which component is generic?  
2. Which component is invoice-specific?  
3. Where is `InvoiceForm` passed as a prop?  
4. What does `CreateItem` calculate generically?  
5. What would break if `InvoiceForm` changed its item field names?  
How to self-grade: Strong answers cite `CreateInvoiceModule`, `CreateItem`, and `InvoiceForm`.
**Connects To:** Mission 17 because composition relies on implicit contracts; Mission 20 because broken contracts create bugs.

### Mission 17: TypeScript's Hidden Work
**Tier:** Mid-Level  
**Time Estimate:** 35 minutes  
**Goal:** Identify what TypeScript would catch here.
**The Concept:** TypeScript is the assistant who says, "That invoice field does not exist" before the customer sees the bug. This repo relies on runtime discovery instead.
**Design Intent Before You Read the Code:** Look for undefined identifiers, stringly typed entities, and implicit payload shapes.
**Find It In The Code:** Open `frontend/src/router/routes.jsx:72-115`, `frontend/src/redux/erp/actions.js:98-118`, `backend/src/setup/setup.js:54-67`, `frontend/jsconfig.json:1-9`.

```jsx
// frontend/src/router/routes.jsx:72-115
{ path: '/quote', element: <Quote /> }          // Undefined identifier in this file.
{ path: '/payment/mode', element: <PaymentMode /> } // Undefined identifier in this file.
{ path: '/taxes', element: <Taxes /> }          // Undefined identifier in this file.
```

**The Aha Moment:** Without compile-time checks, route and model gaps become runtime surprises.
**Socratic Checkpoint:**  
1. Which undefined route identifiers would TS catch?  
2. Which missing backend models would static tooling reveal earlier?  
3. Where are entity names plain strings?  
4. What shape does `recordPayment` expect but not type-check?  
5. What is the first TS migration target you would choose?  
How to self-grade: Strong answers cite route gaps, setup gaps, entity strings, and payment payload construction.
**Connects To:** Mission 23 because tests compensate for missing static checks; Mission 24 because history often explains why gaps exist.

