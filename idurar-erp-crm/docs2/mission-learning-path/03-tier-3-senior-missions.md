# Tier 3 Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions
**Tier:** Senior  
**Time Estimate:** 45 minutes  
**Goal:** Infer why the system is shaped around dynamic CRUD and reusable ERP modules.
**The Concept:** Architecture is the history of tradeoffs. In this ERP, the team optimized for fast entity screens: customers, invoices, payments, settings.
**Design Intent Before You Read the Code:** Identify what the abstraction makes easy and what it hides.
**Find It In The Code:** Open `backend/src/models/utils/index.js:22-47`, `backend/src/controllers/appControllers/index.js:12-33`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:41-215`, `frontend/src/pages/Invoice/index.jsx:93-104`.

```js
// backend/src/models/utils/index.js:34-46
const entity = fileNameWithoutExtension.toLowerCase(); // Model file -> URL entity.
controllerName = fileNameLowerCaseFirstChar + 'Controller'; // Model file -> controller name.
routesList.push({ entity, modelName, controllerName }); // One convention powers routes.
```

**The Aha Moment:** The codebase bets on convention so new entities can be built quickly.
**Socratic Checkpoint:**  
1. What entity workflow does the architecture optimize?  
2. What does dynamic CRUD reduce?  
3. What does dynamic CRUD make harder to catch?  
4. Why do invoices override generic CRUD?  
5. What convention would you document first for new engineers?  
How to self-grade: Strong answers cite dynamic discovery, custom controller override, and invoice money rules.
**Connects To:** Mission 19 because tradeoffs create bug classes; Mission 24 because history explains conventions.

### Mission 19: Find the Bugs Before They Happen
**Tier:** Senior  
**Time Estimate:** 50 minutes  
**Goal:** Predict likely failures from code shape.
**The Concept:** Senior debugging starts before a bug report. You look for mismatched contracts, missing boundaries, and silent assumptions.
**Design Intent Before You Read the Code:** Find problems that would become user-visible: broken navigation, failed setup, stale state, security leaks.
**Find It In The Code:** Open `frontend/src/router/routes.jsx:72-115`, `backend/src/setup/setup.js:54-67`, `backend/src/handlers/errorHandlers.js:66-77`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`.

```js
// backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41
Model.find({
  removed: false,
  [filter]: equal, // User-controlled key enters Mongo query.
  ...fields,
})
```

**The Aha Moment:** Bugs cluster where contracts are implicit and input is trusted.
**Socratic Checkpoint:**  
1. What route will crash first?  
2. What setup line can fail on a clean clone?  
3. What error response leaks too much?  
4. What query construction is risky?  
5. Which one would you fix first and why?  
How to self-grade: Strong answers prioritize setup and security based on user/blocking impact.
**Connects To:** Mission 20 because prediction becomes bug scenarios; Mission 22 because security is a bug class.

### Mission 20: The Bug Injection Challenge
**Tier:** Senior  
**Time Estimate:** 45 minutes  
**Goal:** Describe realistic bugs without changing production code.
**The Concept:** A bug injection exercise is a fire drill. You write symptoms and tests, then prove you know where the fault would live.
**Design Intent Before You Read the Code:** Bugs should be user-visible and tied to exact code paths.
**Find It In The Code:** Open `frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:26-58`, `backend/src/controllers/appControllers/paymentController/create.js:30-77`, `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:10-23`.

```jsx
// frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:26-31
const [maxAmount, setMaxAmount] = useState(0);
useEffect(() => {
  const { credit, total, discount } = currentInvoice; // If currentInvoice shape changes, max amount breaks.
  setMaxAmount(calculate.sub(calculate.sub(total, discount), credit));
}, [currentInvoice]);
```

**The Aha Moment:** Good bug scenarios name symptom, trigger, expected result, and suspected file.
**Socratic Checkpoint:**  
1. What happens if invoice credit is undefined?  
2. What happens if a payment amount exceeds max?  
3. What happens if invoice items array is empty?  
4. What happens if current invoice is not in list cache?  
5. What test would catch each?  
How to self-grade: Strong answers tie each symptom to frontend max calculation, backend max enforcement, Joi validation, and `erp.read` fallback.
**Connects To:** Mission 23 because bug scenarios become tests; Mission 21 because some bugs are performance bugs.

### Mission 21: Performance X-Ray
**Tier:** Senior  
**Time Estimate:** 45 minutes  
**Goal:** Find performance risks and propose concrete fixes.
**The Concept:** Performance is throughput at the invoice counter: fewer duplicate requests, bounded pages, and responsive layout changes.
**Design Intent Before You Read the Code:** Look for unbounded queries, effects that fetch too often, and cleanup that does not cancel work.
**Find It In The Code:** Open `backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:1-7`, `frontend/src/modules/ErpPanelModule/DataTable.jsx:162-168`, `frontend/src/hooks/useResponsive.jsx:40-68`.

```js
// backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:2-4
const page = req.query.page || 1;        // No minimum normalization.
const limit = parseInt(req.query.items) || 10; // No maximum cap.
const skip = page * limit - limit;
```

**The Aha Moment:** Performance risks often look like missing limits, not slow algorithms.
**Socratic Checkpoint:**  
1. Where should page size be capped?  
2. Which abort cleanup is not wired to Axios?  
3. How does responsive state notify subscribers?  
4. What list query would you load-test?  
5. What metric would show improvement?  
How to self-grade: Strong answers cite pagination, abort controller, responsive subscriptions, and table list endpoints.
**Connects To:** Mission 22 because limits also reduce abuse; Mission 23 because performance fixes need regression tests.

### Mission 22: The Security Audit
**Tier:** Senior  
**Time Estimate:** 55 minutes  
**Goal:** Audit auth, error handling, and query inputs.
**The Concept:** Security is the locked file cabinet. The clerk may be friendly, but invoices and admin sessions need firm boundaries.
**Design Intent Before You Read the Code:** Verify authentication, session storage, input validation, and error disclosure.
**Find It In The Code:** Open `frontend/src/request/request.js:16-25`, `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:11-58`, `backend/src/handlers/errorHandlers.js:66-77`, `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`.

```js
// frontend/src/request/request.js:16-25
axios.defaults.withCredentials = true; // Cookies allowed, but token below is header-based.
const auth = storePersist.get('auth'); // Persisted browser storage.
if (auth) {
  axios.defaults.headers.common['Authorization'] = `Bearer ${auth.current.token}`; // JWT header.
}
```

**The Aha Moment:** Auth can be functionally correct while still having storage and disclosure risks.
**Socratic Checkpoint:**  
1. Where is the JWT stored and sent?  
2. Where does backend verify the JWT?  
3. What does production error handler leak?  
4. What query field should be allowlisted?  
5. What is the safest first security PR?  
How to self-grade: Strong answers cite token inclusion, JWT middleware, production errors, invoice filter, and propose small hardening PRs.
**Connects To:** Mission 23 because security behavior should be tested; Mission 24 because auth design often evolves over commits.

### Mission 23: Write the Test That Doesn't Exist
**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** Design a test for the riskiest untested behavior.
**The Concept:** Tests are receipts: they prove the invoice counter did what it claims. The payment flow needs receipts.
**Design Intent Before You Read the Code:** Test backend authority, not just frontend convenience.
**Find It In The Code:** Open `frontend/package.json:30-35`, `backend/package.json:8-15`, `backend/src/controllers/appControllers/paymentController/create.js:59-77`, `backend/src/models/appModels/Invoice.js:118-136`.

```js
// backend/src/controllers/appControllers/paymentController/create.js:66-77
const invoiceUpdate = await Invoice.findOneAndUpdate(
  { _id: req.body.invoice },
  {
    $push: { payment: paymentId.toString() }, // Link payment to invoice.
    $inc: { credit: amount },                 // Increase paid amount.
    $set: { paymentStatus: paymentStatus },   // Update derived status.
  },
  { new: true, runValidators: true }
).exec();
```

**The Aha Moment:** The missing high-value test is "payment changes invoice credit and status correctly."
**Socratic Checkpoint:**  
1. What behavior should be tested first?  
2. Why is backend test more important than frontend test here?  
3. What package scripts are missing?  
4. What fixture data do you need?  
5. What assertion proves the invoice changed?  
How to self-grade: Strong answers cite missing scripts, payment controller update, invoice schema, and test fixture needs.
**Connects To:** Mission 24 because tests preserve behavior through future commits.

### Mission 24: The Git History Tells a Story
**Tier:** Senior  
**Time Estimate:** 35 minutes  
**Goal:** Learn to infer evolution from likely commit messages and current code.
**The Concept:** Git history is archaeology. Current architecture shows layers of decisions: generic CRUD, custom invoice rules, auth hardening, settings boot, and incomplete quote/tax paths.
**Design Intent Before You Read the Code:** Read current code as evidence of earlier priorities and unfinished migrations.
**Find It In The Code:** Open `backend/src/controllers/appControllers/index.js:12-33`, `backend/src/controllers/appControllers/invoiceController/index.js:1-20`, `frontend/src/apps/ErpApp.jsx:36-90`, `frontend/src/router/routes.jsx:72-115`.

```js
// backend/src/controllers/appControllers/invoiceController/index.js:12-18
methods.mail = sendMail;      // Custom mail behavior.
methods.create = create;      // Override generic create.
methods.update = update;      // Override generic update.
methods.delete = remove;      // Override generic delete.
methods.summary = summary;    // Override generic summary.
methods.list = paginatedList; // Override generic list.
methods.read = read;          // Override generic read.
```

**The Aha Moment:** Custom overrides are fossil records of where the product needed more than CRUD.
**Socratic Checkpoint:**  
1. What commit probably introduced generic CRUD?  
2. What commit probably introduced invoice overrides?  
3. What commit probably introduced settings boot gating?  
4. What commit might have left quote/tax paths incomplete?  
5. How would history guide refactoring?  
How to self-grade: Strong answers infer change intent from current override and route patterns.
**Connects To:** Mission 25 because writing docs turns archaeology into shared team memory.

