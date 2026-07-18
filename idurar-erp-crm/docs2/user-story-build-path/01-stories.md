# User Stories

## Story 1: Show Invoice Status As A Colored Tag
**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Skills You'll Practice:** component reuse, table rendering, localization  
**The Story:** As an admin, I want invoice status to be visually distinct so that I can scan invoice lists faster.
**Acceptance Criteria:**
- [ ] The invoice table status column renders a colored tag instead of raw text.
- [ ] Existing invoice statuses still display translated labels.
- [ ] The invoice list route `/invoice` still loads.
**Files You'll Likely Touch:**  
`frontend/src/pages/Invoice/index.jsx` - defines invoice table columns and status column (`frontend/src/pages/Invoice/index.jsx:21-82`).  
`frontend/src/components/Tag/index.jsx` - existing `StatusTag` component maps status to tag color (`frontend/src/components/Tag/index.jsx:4-19`).
**High-Level Implementation Plan:**
1. In `frontend/src/pages/Invoice/index.jsx`, import the existing status tag component.
2. Modify the `status` column render function to return the status tag.
3. Manually load `/invoice` and confirm the column still sorts visually in the table.
**Tips:**
- The status column is currently `dataIndex: 'status'` with no render function (`frontend/src/pages/Invoice/index.jsx:74-77`).
- Keep the render logic in the column; do not change backend data.
- Check whether `StatusTag` color options align with invoice schema statuses (`backend/src/models/appModels/Invoice.js:148-152`).
**What Could Go Wrong:**
- Some invoice statuses may fall into the default red color because `StatusTag` was written for a different status set (`frontend/src/components/Tag/index.jsx:6-15`).
- Import alias mistakes can break Vite resolution; existing imports use `@/` (`frontend/src/pages/Invoice/index.jsx:3-7`).
**Stretch Goal:** Add missing color mappings for `pending`, `refunded`, `cancelled`, and `on hold`.
**Connects To:** Story 2 because both require safe changes to invoice table display.

## Story 2: Add Created-By To Invoice List
**Difficulty:** Easy  
**Estimated Time:** 1.5 hours  
**Skills You'll Practice:** table columns, populated backend fields, defensive rendering  
**The Story:** As an admin, I want to see who created each invoice so that ownership is clear.
**Acceptance Criteria:**
- [ ] The invoice table includes a "Created By" column.
- [ ] The column shows the admin name when present.
- [ ] The table does not crash when `createdBy` is missing.
**Files You'll Likely Touch:**  
`frontend/src/pages/Invoice/index.jsx` - add a table column (`frontend/src/pages/Invoice/index.jsx:21-82`).  
`backend/src/controllers/appControllers/invoiceController/paginatedList.js` - already populates `createdBy` name for invoice list (`backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-33`).
**High-Level Implementation Plan:**
1. Confirm invoice list backend returns `createdBy.name` from the populate call.
2. Add a column in `dataTableColumns` using nested data index or a render function.
3. Use a fallback such as blank text when `createdBy` is unavailable.
**Tips:**
- The invoice model references `createdBy` as an Admin (`backend/src/models/appModels/Invoice.js:9-10`).
- The invoice read controller also populates `createdBy` (`backend/src/controllers/appControllers/invoiceController/read.js:5-12`).
- Follow the existing client nested column style (`frontend/src/pages/Invoice/index.jsx:26-29`).
**What Could Go Wrong:**
- Using a nested path without a fallback can render empty values for older records.
- If you add the column in the wrong place, the table can feel cluttered on small screens.
**Stretch Goal:** Show "Unknown" through localization instead of a hardcoded fallback.
**Connects To:** Story 3 because both improve customer-facing invoice clarity.

## Story 3: Improve Empty Invoice Item Validation Message
**Difficulty:** Easy  
**Estimated Time:** 1.5 hours  
**Skills You'll Practice:** form validation, backend validation, user feedback  
**The Story:** As an admin creating invoices, I want a clear error when I forget line items so that I know how to fix the invoice.
**Acceptance Criteria:**
- [ ] Submitting an invoice with no items shows a clear validation message.
- [ ] Backend validation prevents empty item arrays.
- [ ] Existing valid invoice creation still works.
**Files You'll Likely Touch:**  
`backend/src/controllers/appControllers/invoiceController/schemaValidate.js` - validates invoice payload (`backend/src/controllers/appControllers/invoiceController/schemaValidate.js:1-26`).  
`frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx` - renders `Form.List` item rows (`frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:193-212`).
**High-Level Implementation Plan:**
1. Update Joi array validation to require at least one item.
2. Add or confirm frontend form behavior creates at least one row by default.
3. Manually test by removing all item rows and submitting.
**Tips:**
- `InvoiceForm` auto-clicks the add-field button on mount (`frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:60-64`).
- The backend schema currently requires an array but does not require `.min(1)` (`backend/src/controllers/appControllers/invoiceController/schemaValidate.js:10-23`).
- Backend validation should remain authoritative.
**What Could Go Wrong:**
- Frontend-only validation can be bypassed by direct API calls.
- Joi message text may be technical unless customized.
**Stretch Goal:** Add a frontend `Form.List` rule for friendlier immediate feedback.
**Connects To:** Story 4 because invoice item behavior prepares you to read invoice totals.

## Story 4: Add Invoice Balance Due To The Table
**Difficulty:** Medium-Easy  
**Estimated Time:** 2 hours  
**Skills You'll Practice:** derived UI values, money formatting, table rendering  
**The Story:** As an admin, I want to see balance due on each invoice so that I know what is still owed.
**Acceptance Criteria:**
- [ ] Invoice table shows a "Balance Due" column.
- [ ] Balance due equals `total - discount - credit`.
- [ ] The value uses the existing money formatter and invoice currency.
**Files You'll Likely Touch:**  
`frontend/src/pages/Invoice/index.jsx` - table column definitions and money formatting (`frontend/src/pages/Invoice/index.jsx:44-73`).  
`frontend/src/utils/calculate.js` - existing precise currency math helpers (`frontend/src/utils/calculate.js:1-18`).  
`backend/src/models/appModels/Invoice.js` - invoice fields for total, credit, discount, currency (`backend/src/models/appModels/Invoice.js:108-125`).
**High-Level Implementation Plan:**
1. Import `calculate` if needed in `frontend/src/pages/Invoice/index.jsx`.
2. Add a new column after paid/credit.
3. Compute balance with existing helper functions.
4. Format with `moneyFormatter({ amount, currency_code: record.currency })`.
**Tips:**
- Existing total and paid columns show the right alignment style (`frontend/src/pages/Invoice/index.jsx:44-73`).
- Payment creation uses the same balance formula on the backend (`backend/src/controllers/appControllers/paymentController/create.js:24-38`).
- Keep derived display read-only; do not persist a new field.
**What Could Go Wrong:**
- Forgetting discount makes partially discounted invoices look wrong.
- Direct subtraction can create floating point issues; use `calculate`.
**Stretch Goal:** Highlight zero balance due for paid invoices.
**Connects To:** Story 5 because the dashboard also summarizes money and recent activity.

## Story 5: Add Customer Email To Customer List
**Difficulty:** Medium-Easy  
**Estimated Time:** 2 hours  
**Skills You'll Practice:** CRUD config, schema reading, table UI  
**The Story:** As an admin, I want customer email visible in the customer list so that I can contact clients faster.
**Acceptance Criteria:**
- [ ] Customer list includes an email column.
- [ ] Empty email values do not break the table.
- [ ] Existing customer create/update behavior still works.
**Files You'll Likely Touch:**  
`frontend/src/pages/Customer/index.jsx` - customer page config.  
`frontend/src/pages/Customer/config.js` - likely customer field/table configuration.  
`backend/src/models/appModels/Client.js` - confirms client `email` field exists (`backend/src/models/appModels/Client.js:13-22`).
**High-Level Implementation Plan:**
1. Inspect the customer page/config files to find current columns.
2. Add an email column using the same pattern as nearby fields.
3. Load `/customer` and confirm list rendering.
**Tips:**
- The backend client model already has `email: String` (`backend/src/models/appModels/Client.js:17-21`).
- Customer routes are generated from the `Client` model through dynamic entity routing (`backend/src/models/utils/index.js:22-47`).
- Follow the existing page config style rather than creating a new table component.
**What Could Go Wrong:**
- Customer config may use generic `fields` rather than explicit columns, so read before editing.
- Search may not include email unless separately configured.
**Stretch Goal:** Add email to customer search fields if the page supports search config.
**Connects To:** Story 6 because customer data becomes useful in new API/state behavior.

## Story 6: Add Invoice Search By Number
**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** query parameters, Redux list actions, backend list filtering  
**The Story:** As an admin, I want to search invoices by number so that I can find a specific invoice quickly.
**Acceptance Criteria:**
- [ ] Invoice list search can match invoice number.
- [ ] Existing client filter behavior still works.
- [ ] Backend handles invalid search input safely.
**Files You'll Likely Touch:**  
`frontend/src/pages/Invoice/index.jsx` - search config currently targets client name (`frontend/src/pages/Invoice/index.jsx:15-20`).  
`frontend/src/modules/ErpPanelModule/DataTable.jsx` - filter behavior for ERP tables (`frontend/src/modules/ErpPanelModule/DataTable.jsx:170-172`).  
`backend/src/controllers/appControllers/invoiceController/paginatedList.js` - invoice list query construction (`backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-67`).  
`frontend/src/redux/erp/actions.js` - list action passes options (`frontend/src/redux/erp/actions.js:36-68`).
**High-Level Implementation Plan:**
1. Decide whether search by invoice number should use the existing client auto-complete or a new text input.
2. If adding text search, extend `DataTable` carefully so existing entity filters still work.
3. Update backend invoice list query to handle number search safely.
4. Test by searching existing invoice numbers and clearing search.
**Tips:**
- Generic CRUD `paginatedList` already supports `fields` and `q` regex search (`backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:8-17`).
- Invoice-specific pagination does not include the generic filter guard (`backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-41`).
- Keep query parameters explicit and encode user input.
**What Could Go Wrong:**
- Regex against numeric fields may not behave as expected.
- Mixing client filter and number search can create confusing query combinations.
**Stretch Goal:** Add a segmented search mode: client vs invoice number.
**Connects To:** Story 7 because both add state/API behavior beyond simple display.

## Story 7: Add A Payment Success Summary
**Difficulty:** Medium  
**Estimated Time:** 3.5 hours  
**Skills You'll Practice:** Redux success state, navigation timing, derived values  
**The Story:** As an admin recording a payment, I want a success summary so that I know how the invoice balance changed.
**Acceptance Criteria:**
- [ ] After payment succeeds, the UI shows payment amount and remaining balance before navigating or on the destination page.
- [ ] The payment form still resets after success.
- [ ] The invoice list reload still occurs.
**Files You'll Likely Touch:**  
`frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx` - success effect and submit payload (`frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:33-58`).  
`frontend/src/redux/erp/actions.js` - `recordPayment` success payload handling (`frontend/src/redux/erp/actions.js:98-118`).  
`frontend/src/redux/erp/reducer.js` - recordPayment state bucket (`frontend/src/redux/erp/reducer.js:44-49`).  
`backend/src/controllers/appControllers/paymentController/create.js` - payment response result (`backend/src/controllers/appControllers/paymentController/create.js:79-83`).
**High-Level Implementation Plan:**
1. Inspect what `paymentController.create` returns today.
2. Decide whether frontend can compute remaining balance from current invoice and form amount or needs backend response changes.
3. Add UI feedback using existing Ant Design patterns.
4. Preserve reset, list refresh, and navigation behavior.
**Tips:**
- `RecordPayment` currently navigates immediately on `isSuccess` (`frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:33-40`).
- The thunk stores `data.result` in `recordPayment.result` (`frontend/src/redux/erp/actions.js:109-114`).
- Backend currently returns the payment document, not the updated invoice (`backend/src/controllers/appControllers/paymentController/create.js:79-83`).
**What Could Go Wrong:**
- Immediate navigation may hide your summary.
- Computing from stale invoice state can show the wrong balance if backend rejects or adjusts payment.
**Stretch Goal:** Return the updated invoice in the backend response and update frontend state from that.
**Connects To:** Story 8 because returning richer backend data is a full-stack change.

## Story 8: Add Payment Method Persistence
**Difficulty:** Hard  
**Estimated Time:** 6 hours  
**Skills You'll Practice:** schema changes, setup gaps, form fields, backend controllers  
**The Story:** As an admin, I want each payment to store its payment method so that reports can distinguish cash, wire, and other methods.
**Acceptance Criteria:**
- [ ] Payment records persist a payment method value.
- [ ] Payment form submits the selected method.
- [ ] Existing payment creation still updates invoice credit/status.
- [ ] Setup or fallback data supports at least one default payment method.
**Files You'll Likely Touch:**  
`frontend/src/forms/PaymentForm.jsx` - already renders `paymentMode` select (`frontend/src/forms/PaymentForm.jsx:54-70`).  
`backend/src/models/appModels/Payment.js` - payment schema currently has no `paymentMode` field (`backend/src/models/appModels/Payment.js:3-57`).  
`backend/src/controllers/appControllers/paymentController/create.js` - creates payment from request body (`backend/src/controllers/appControllers/paymentController/create.js:39-53`).  
`backend/src/setup/setup.js` - currently tries to seed missing `PaymentMode` model (`backend/src/setup/setup.js:54-67`).  
`backend/src/controllers/appControllers/paymentController/update.js` - update path references `paymentMode` in update data (`backend/src/controllers/appControllers/paymentController/update.js:52`).
**High-Level Implementation Plan:**
1. Decide whether payment method is a string field or a restored `PaymentMode` model reference.
2. Update `Payment` schema to persist the chosen value.
3. Ensure `PaymentForm` output matches backend schema.
4. Fix setup seed path if restoring `PaymentMode`.
5. Test record-payment and payment update flows.
**Tips:**
- `PaymentForm` already assumes a `paymentMode` entity exists (`frontend/src/forms/PaymentForm.jsx:54-70`).
- Backend setup assumes `PaymentMode` exists, but the model file is missing (`backend/src/setup/setup.js:54-67`).
- Keep invoice credit/status logic unchanged while adding method persistence.
**What Could Go Wrong:**
- Restoring `PaymentMode` partially can create new broken routes if frontend pages remain missing.
- Storing an object where the schema expects a string/ObjectId can fail validation.
**Stretch Goal:** Add a payment method column to the payment list.
**Connects To:** Story 9 because both require schema-backed full-stack work.

## Story 9: Add Invoice Overdue Detection
**Difficulty:** Hard  
**Estimated Time:** 7 hours  
**Skills You'll Practice:** date logic, backend derived fields, UI display, list filtering  
**The Story:** As an admin, I want overdue invoices marked automatically so that I can prioritize collections.
**Acceptance Criteria:**
- [ ] Invoices past `expiredDate` and not fully paid are marked overdue.
- [ ] Invoice list displays overdue status clearly.
- [ ] Paid invoices are not marked overdue.
- [ ] Backend remains the source of truth for overdue logic.
**Files You'll Likely Touch:**  
`backend/src/models/appModels/Invoice.js` - existing `isOverdue` field (`backend/src/models/appModels/Invoice.js:137-140`).  
`backend/src/controllers/appControllers/invoiceController/paginatedList.js` - invoice list retrieval (`backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-67`).  
`backend/src/controllers/appControllers/invoiceController/read.js` - invoice read retrieval (`backend/src/controllers/appControllers/invoiceController/read.js:5-28`).  
`frontend/src/pages/Invoice/index.jsx` - invoice table display (`frontend/src/pages/Invoice/index.jsx:21-82`).  
`frontend/src/components/Tag/index.jsx` - potential status tag reuse (`frontend/src/components/Tag/index.jsx:4-19`).  
`backend/src/controllers/appControllers/paymentController/create.js` - payment can make overdue invoice paid (`backend/src/controllers/appControllers/paymentController/create.js:59-77`).
**High-Level Implementation Plan:**
1. Define overdue as `expiredDate < now` and `paymentStatus !== 'paid'`.
2. Add backend logic to compute or update `isOverdue` during invoice list/read and payment update paths.
3. Add an invoice table column or status display for overdue.
4. Test unpaid expired, partially paid expired, and paid expired invoices.
**Tips:**
- Invoice schema already has `isOverdue` with default false (`backend/src/models/appModels/Invoice.js:137-140`).
- Payment status is updated when payments are recorded (`backend/src/controllers/appControllers/paymentController/create.js:59-77`).
- Use backend date logic so direct API use stays correct.
**What Could Go Wrong:**
- Timezone differences can make "today" invoices look overdue too early.
- Computing overdue only on list can leave read/payment views inconsistent.
**Stretch Goal:** Add a filter to show only overdue invoices.
**Connects To:** Story 10 because overdue filtering/caching raises architectural questions.

## Story 10: Add Cached Dashboard Metrics
**Difficulty:** Expert  
**Estimated Time:** 10+ hours  
**Skills You'll Practice:** architecture design, caching, invalidation, backend aggregation, frontend state  
**The Story:** As an admin, I want dashboard totals to load quickly and stay reasonably fresh so that I can monitor the business without waiting on repeated expensive queries.
**Acceptance Criteria:**
- [ ] Dashboard metrics are served by a backend endpoint with a documented cache strategy.
- [ ] Cache invalidates or refreshes after invoice/payment changes.
- [ ] Frontend dashboard uses the cached endpoint.
- [ ] Security still requires a valid admin token.
- [ ] The design explains stale-data tolerance and invalidation triggers.
**Files You'll Likely Touch:**  
`frontend/src/modules/DashboardModule/index.jsx` - dashboard fetches stats and recent data (`frontend/src/modules/DashboardModule/index.jsx:39-56`, `frontend/src/modules/DashboardModule/index.jsx:138-188`).  
`frontend/src/hooks/useOnFetch.jsx` - dashboard uses async fetch state (`frontend/src/hooks/useOnFetch.jsx:3-22`).  
`frontend/src/request/request.js` - generic `summary`/`get` patterns (`frontend/src/request/request.js:206-214`, `frontend/src/request/request.js:253-272`).  
`backend/src/controllers/appControllers/invoiceController/summary.js` - existing invoice summary behavior.  
`backend/src/controllers/appControllers/paymentController/summary.js` - existing payment summary behavior.  
`backend/src/routes/appRoutes/appApi.js` - summary route generation (`backend/src/routes/appRoutes/appApi.js:17-18`).  
`backend/src/controllers/appControllers/invoiceController/create.js` - invoice creation invalidation trigger (`backend/src/controllers/appControllers/invoiceController/create.js:50-71`).  
`backend/src/controllers/appControllers/paymentController/create.js` - payment creation invalidation trigger (`backend/src/controllers/appControllers/paymentController/create.js:66-83`).  
`backend/package.json` - backend already depends on `node-cache` (`backend/package.json:36-38`).
**High-Level Implementation Plan:**
1. Inspect existing dashboard and summary controllers.
2. Design the cache key: metrics should probably include currency or admin scope.
3. Implement a backend cached dashboard endpoint or cached summary service.
4. Invalidate cache when invoices or payments are created, updated, or deleted.
5. Update dashboard frontend to call the cached endpoint.
6. Document stale-data behavior in code comments or a small doc note.
7. Test login-protected dashboard load and post-payment refresh.
**Tips:**
- `node-cache` is already installed, so avoid adding a new dependency first (`backend/package.json:36-38`).
- App API summary routes already exist for entities (`backend/src/routes/appRoutes/appApi.js:17-18`).
- Payment creation mutates invoice totals, so payment writes must invalidate invoice/dashboard metrics (`backend/src/controllers/appControllers/paymentController/create.js:66-77`).
- Keep auth protection by mounting under `/api` behind existing middleware (`backend/src/app.js:39-43`).
**What Could Go Wrong:**
- Cache keys that ignore currency or user/admin scope can show wrong totals.
- Invalidating only invoice creates but not payments leaves stale paid/unpaid metrics.
- A new public route could accidentally bypass auth if mounted outside protected `/api`.
**Stretch Goal:** Add a visible "last updated" timestamp to the dashboard.
**Connects To:** This is the capstone story because it requires architecture judgment across frontend, backend, state, performance, and security.

