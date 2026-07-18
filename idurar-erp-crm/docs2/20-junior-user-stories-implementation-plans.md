# 20 Junior User Stories With Senior Implementation Plans

This document is written as if a product owner is assigning practical tickets to a junior engineer, and a senior engineer is giving the implementation map. Each story is scoped to this repository and points to real files, components, controllers, and data flows.

Use the plans in order when possible. The first stories are safe UI and configuration work. Later stories touch Redux, backend controllers, Mongoose models, setup scripts, and architecture concerns.

## Story 1: Color-Code Invoice Status In The Invoice List

**Product Story:** As an admin, I want invoice statuses to appear as colored tags so that I can scan the invoice list quickly.

**Why This Matters:** Invoices are the main workflow in this ERP. A raw `draft` or `sent` text value is easy to miss when a user is deciding what needs action.

**Acceptance Criteria:**
- The invoice status column renders a colored Ant Design tag.
- The label still uses the existing translation system.
- The `/invoice` page still loads and lists invoices.
- Unknown statuses have a safe fallback color.

**Code Tour:**
- `frontend/src/pages/Invoice/index.jsx:9-104` defines the invoice page, `entity = 'invoice'`, table columns, search config, and passes config to `InvoiceDataTableModule`.
- `frontend/src/pages/Invoice/index.jsx:74-81` currently renders `status` and `paymentStatus` as raw table values.
- `frontend/src/components/Tag/index.jsx:1-19` contains an existing `StatusTag` helper using Ant Design `Tag` and `useLanguage`.
- `backend/src/models/appModels/Invoice.js:148-152` defines the backend invoice `status` enum values.

**Senior Implementation Plan:**
1. Open `frontend/src/pages/Invoice/index.jsx` and locate the `dataTableColumns` array.
2. Import the existing `StatusTag` from `@/components/Tag` near the other imports.
3. Add a `render` function to the invoice status column:
   - Input should be the current status value.
   - Output should be `<StatusTag status={status} />`.
4. Compare the status values in the backend enum to the existing color function. The tag currently knows `draft`, `sent`, `accepted`, and `expired`, while invoices know `draft`, `pending`, `sent`, `refunded`, `cancelled`, and `on hold`.
5. Update `frontend/src/components/Tag/index.jsx` if needed so invoice statuses have intentional colors.
6. Keep this frontend-only. Do not touch the schema or controller for a display-only change.

**Manual QA:**
- Visit `/invoice`.
- Confirm the status column renders tags.
- Confirm the payment status column is unchanged.
- Confirm browser console has no import or render errors.

**Mentor Notes:** This is a perfect first ticket because it teaches the page-config pattern without changing data. The key lesson is that table behavior lives in the page config, while reusable table mechanics live deeper in modules.

## Story 2: Add Balance Due To The Invoice Table

**Product Story:** As an admin, I want to see the remaining balance due for each invoice so that I know what still needs to be collected.

**Why This Matters:** The system already shows total and paid amount, but users should not have to mentally calculate `total - discount - credit`.

**Acceptance Criteria:**
- The invoice list has a "Balance Due" column.
- Balance due is calculated as `total - discount - credit`.
- The value uses the existing money formatter and invoice currency.
- Fully paid invoices show zero balance.

**Code Tour:**
- `frontend/src/pages/Invoice/index.jsx:44-73` shows existing money columns for `total` and `credit`.
- `frontend/src/utils/calculate.js:1-18` provides precise currency math helpers.
- `backend/src/models/appModels/Invoice.js:108-125` defines `total`, `credit`, `discount`, and `currency`.
- `backend/src/controllers/appControllers/paymentController/create.js:24-38` uses the same balance concept to limit payment amount.

**Senior Implementation Plan:**
1. Open `frontend/src/pages/Invoice/index.jsx`.
2. Import `calculate` from `@/utils/calculate`.
3. Add a new table column after the `paid` column.
4. In the column `render`, compute:
   - `afterDiscount = calculate.sub(record.total, record.discount || 0)`
   - `balance = calculate.sub(afterDiscount, record.credit || 0)`
5. Format with the same `moneyFormatter({ amount, currency_code: record.currency })` style used by existing columns.
6. Reuse the same `onCell` alignment object from the total/paid columns so money remains right-aligned.
7. Do not persist `balanceDue`; it is derived display data.

**Manual QA:**
- Create or inspect an unpaid invoice and confirm balance equals total minus discount.
- Record a partial payment and confirm balance decreases.
- Confirm paid invoice balance is zero.

**Mentor Notes:** This story teaches derived UI state. The backend remains the source of truth for invoice totals and payment enforcement, but the frontend can safely derive read-only display values.

## Story 3: Add Created-By To The Invoice List

**Product Story:** As an admin, I want to see who created an invoice so that ownership is clear.

**Why This Matters:** When multiple admins eventually use the system, invoice ownership helps with accountability and support.

**Acceptance Criteria:**
- The invoice table includes a "Created By" column.
- It displays the admin name when available.
- It does not crash if the value is missing.

**Code Tour:**
- `backend/src/models/appModels/Invoice.js:9-10` defines `createdBy` as an Admin reference.
- `backend/src/controllers/appControllers/invoiceController/create.js:47-48` sets `createdBy` from `req.admin._id`.
- `backend/src/controllers/appControllers/invoiceController/paginatedList.js:22-33` populates `createdBy` with `name`.
- `frontend/src/pages/Invoice/index.jsx:21-82` defines invoice table columns.

**Senior Implementation Plan:**
1. Confirm the backend already sends `createdBy.name` in list results because `paginatedList` calls `.populate('createdBy', 'name')`.
2. Open `frontend/src/pages/Invoice/index.jsx`.
3. Add a column near the client/date columns:
   - `title: translate('Created By')`
   - Use `dataIndex: ['createdBy', 'name']` or a defensive `render`.
4. Prefer a defensive render if existing data may be incomplete:
   - Return `createdBy?.name || ''`.
5. Do not modify backend unless the field is missing from actual responses during manual testing.

**Manual QA:**
- Load `/invoice`.
- Confirm created-by names appear for invoices created through the app.
- Confirm older records or incomplete data do not crash the table.

**Mentor Notes:** This story teaches you to verify backend shape before adding frontend UI. Never assume a nested field exists just because the schema says it can.

## Story 4: Add Customer Email Search

**Product Story:** As an admin, I want to search customers by email so that I can quickly find the right client.

**Why This Matters:** Customer lookup is a CRM workflow. Email is often more precise than name.

**Acceptance Criteria:**
- Customer search matches by name or email.
- Existing customer list and create/update behavior still work.
- The search field still displays customer names in results.

**Code Tour:**
- `frontend/src/pages/Customer/index.jsx:7-39` creates the customer CRUD page.
- `frontend/src/pages/Customer/index.jsx:10-13` currently sets `searchFields: 'name'`.
- `frontend/src/pages/Customer/config.js:1-18` defines form fields including `email`.
- `backend/src/models/appModels/Client.js:13-22` defines `name`, `phone`, `country`, `address`, and `email`.
- `frontend/src/components/SearchItem/index.jsx` should be inspected before implementation because it consumes `searchConfig`.

**Senior Implementation Plan:**
1. Open `frontend/src/pages/Customer/index.jsx` and find `searchConfig`.
2. Change `searchFields` from `'name'` to `'name,email'` if the search component and backend use comma-separated fields.
3. Verify the generic backend search/list behavior:
   - Generic paginated list splits `req.query.fields` on commas (`backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:8-17`).
   - Generic search defaults to `name` but can accept `fields` options.
4. Inspect `frontend/src/components/SearchItem/index.jsx` to confirm whether it uses `searchConfig.searchFields`.
5. If `SearchItem` does not pass fields through, update it in the smallest possible way so it sends the configured fields.
6. Keep the customer form unchanged because `email` already exists in config and model.

**Manual QA:**
- Create or find a customer with an email address.
- Search by part of the email.
- Search by name to ensure the old behavior still works.

**Mentor Notes:** This story trains you to follow configuration from page to reusable component to backend query. The change might be one line, but the responsible work is verifying the whole path.

## Story 5: Make Customer Phone Optional But Better Formatted

**Product Story:** As an admin, I want phone numbers to be optional but clearly formatted so that customer records can be saved even when only email is known.

**Why This Matters:** Early CRM records are often incomplete. The system should accept partial customer information without making the UI feel sloppy.

**Acceptance Criteria:**
- Customer phone remains optional.
- The phone input keeps a helpful placeholder.
- Customer create/update still works with only name and email.
- The UI does not add backend validation that the model does not enforce.

**Code Tour:**
- `frontend/src/pages/Customer/config.js:1-18` marks customer fields and uses `phone` type for phone.
- `frontend/src/forms/DynamicForm/index.jsx:297-305` maps `phone` fields to an Ant Design `Input` with placeholder `+1 123 456 789`.
- `backend/src/models/appModels/Client.js:13-22` requires only `name`; phone and email are plain strings.
- `frontend/src/pages/Customer/index.jsx:32-38` passes `DynamicForm` into `CrudModule`.

**Senior Implementation Plan:**
1. Inspect `frontend/src/pages/Customer/config.js`; confirm `phone` has no `required: true`.
2. Inspect `DynamicForm` to understand how field type drives input rendering.
3. If the placeholder is acceptable, do not change behavior. Instead, add a clearer label or maxLength in the customer field config.
4. If you add `maxLength`, do it in `config.js` because `DynamicForm` already passes `field.maxLength` to string inputs, but phone currently uses a specific input. If maxLength does not apply to phone, update the phone component carefully.
5. Do not add backend required validation because the schema does not require phone.

**Manual QA:**
- Create a customer with name and email only.
- Create a customer with phone.
- Update an existing customer and clear phone.

**Mentor Notes:** Sometimes a junior ticket is about not overbuilding. Align frontend validation with backend schema unless the product explicitly wants stricter UI rules.

## Story 6: Add Customer Country To Invoice Read View

**Product Story:** As an admin viewing an invoice, I want to see the customer's country so that I have billing context.

**Why This Matters:** Invoices reference clients. The read view should surface useful client details without requiring a separate customer lookup.

**Acceptance Criteria:**
- Invoice read view shows client country when available.
- It does not crash if country is missing.
- No backend change is made unless the data is not present.

**Code Tour:**
- `backend/src/models/appModels/Invoice.js:31-36` references `Client` and uses autopopulate.
- `backend/src/models/appModels/Client.js:13-22` defines `country`.
- `backend/src/controllers/appControllers/invoiceController/read.js:5-28` fetches an invoice by id and populates `createdBy`.
- `frontend/src/pages/Invoice/InvoiceRead.jsx:4-20` creates invoice read config.
- `frontend/src/modules/InvoiceModule/ReadInvoiceModule/index.jsx:13-39` dispatches `erp.read` and renders `ReadItem`.
- `frontend/src/modules/ErpPanelModule/ReadItem.jsx` should be inspected to determine where fields are displayed.

**Senior Implementation Plan:**
1. Open `frontend/src/modules/ErpPanelModule/ReadItem.jsx`.
2. Identify how selected invoice fields are displayed.
3. Confirm `selectedItem.client.country` exists in the API response. Because the invoice model autopopulates client, it likely does.
4. Add a defensive display field for country:
   - Use optional chaining.
   - Show nothing or a translated fallback if missing.
5. Avoid backend changes unless manual API inspection proves `client.country` is absent.

**Manual QA:**
- Open `/invoice/read/:id` for an invoice with a client country.
- Open one with no country.
- Confirm the read page still loads after refresh.

**Mentor Notes:** This story trains read-path debugging. The page may work from Redux cache or from a direct backend fetch; test both.

## Story 7: Show Payment Invoice Number More Clearly

**Product Story:** As an admin reviewing payments, I want the related invoice number to be clearly labeled so that I do not confuse payment number with invoice number.

**Why This Matters:** The payment table currently has two columns titled "Number", one for payment and one for invoice. That is confusing.

**Acceptance Criteria:**
- Payment number column remains.
- Related invoice number column is renamed to "Invoice Number".
- Invoice year remains visible.
- No backend change is required.

**Code Tour:**
- `frontend/src/pages/Payment/index.jsx:19-63` defines payment table columns.
- `frontend/src/pages/Payment/index.jsx:20-24` displays payment `number`.
- `frontend/src/pages/Payment/index.jsx:51-58` displays invoice `number` and `year`.
- `backend/src/models/appModels/Payment.js:20-25` references an invoice and autopopulates it.

**Senior Implementation Plan:**
1. Open `frontend/src/pages/Payment/index.jsx`.
2. Find the second column titled `translate('Number')` with `dataIndex: ['invoice', 'number']`.
3. Change its title to `translate('Invoice Number')`.
4. Check `frontend/src/locale/translation/en_us.js` for whether `invoice_number` or `Invoice Number` exists. If not, add a translation key using the repo's translation style.
5. Keep `dataIndex` unchanged.

**Manual QA:**
- Load `/payment`.
- Confirm two number columns no longer share the same label.
- Confirm invoice values still display.

**Mentor Notes:** Small UX clarity changes matter in business software. In review, always ask whether two nearby fields with similar labels could be misread.

## Story 8: Add Payment Reference To The Payment Table

**Product Story:** As an admin, I want to see a payment reference in the payment list so that I can reconcile payments against bank records.

**Why This Matters:** Payment references are operationally important for finance reconciliation.

**Acceptance Criteria:**
- Payment list includes a "Reference" column.
- Blank references do not break the table.
- Payment record form still accepts reference.

**Code Tour:**
- `frontend/src/forms/PaymentForm.jsx:71-75` already collects `ref` and `description`.
- `backend/src/models/appModels/Payment.js:41-46` defines `ref` and `description`.
- `frontend/src/pages/Payment/index.jsx:19-63` defines the payment table.
- `backend/src/controllers/appControllers/paymentController/create.js:39-53` creates payment records from the submitted body.

**Senior Implementation Plan:**
1. Open `frontend/src/pages/Payment/index.jsx`.
2. Add a column near amount/date or after invoice year:
   - `title: translate('Reference')`
   - `dataIndex: 'ref'`
   - Optional render returning `ref || ''`.
3. Confirm translation exists for `Reference`; the form already uses `translate('Reference')`.
4. Do not change backend because model and form already support the field.

**Manual QA:**
- Record a payment with a reference.
- Load `/payment`.
- Confirm the reference appears.
- Confirm older payments with empty refs render safely.

**Mentor Notes:** This story shows a common full-stack shortcut: before adding a field, verify whether the backend already supports it. Here, the UI list is simply not showing existing data.

## Story 9: Improve Invoice Create Validation For Empty Items

**Product Story:** As an admin creating an invoice, I want a clear error if I remove all item rows so that I understand why the invoice cannot be saved.

**Why This Matters:** An invoice without line items is not useful and can create confusing zero-value records.

**Acceptance Criteria:**
- Backend rejects invoice create/update with no line items.
- Frontend gives a clear validation cue if all item rows are removed.
- Existing valid invoice creation still works.

**Code Tour:**
- `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:193-212` renders invoice items with `Form.List`.
- `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:60-64` auto-adds an item row on mount.
- `backend/src/controllers/appControllers/invoiceController/schemaValidate.js:10-23` requires an items array but does not require at least one item.
- `backend/src/controllers/appControllers/invoiceController/update.js:30-38` explicitly rejects empty items on update.
- `backend/src/controllers/appControllers/invoiceController/create.js:12-23` validates create payload.

**Senior Implementation Plan:**
1. Update Joi schema in `schemaValidate.js` to require `items.min(1)`.
2. Check that both create and update use this schema. They do, so one schema change affects both paths.
3. Add frontend `Form.List` validation only if you can do it cleanly with Ant Design.
4. Do not remove the existing auto-add behavior because it helps new invoices start with one row.
5. Test direct API behavior mentally: frontend validation is helpful, backend validation is mandatory.

**Manual QA:**
- Start creating an invoice.
- Remove all line items.
- Attempt save.
- Confirm a useful error appears and no invoice is created.
- Create a valid invoice with one row.

**Mentor Notes:** Backend validation wins. UI validation improves user experience, but the backend protects data integrity.

## Story 10: Fix Broken Quote Links By Hiding Or Restoring Them

**Product Story:** As an admin, I do not want navigation links that crash the app, so that the ERP feels reliable.

**Why This Matters:** Navigation currently exposes quote routes even though the inspected route file references undefined quote components.

**Acceptance Criteria:**
- Clicking visible navigation links does not crash the frontend.
- Quote links are either restored with valid imports/files or hidden until implemented.
- Dashboard does not call quote APIs if quote backend support is absent.

**Code Tour:**
- `frontend/src/apps/Navigation/NavigationContainer.jsx:68-72` shows a quote navigation item.
- `frontend/src/router/routes.jsx:72-87` uses `Quote`, `QuoteCreate`, `QuoteRead`, and `QuoteUpdate` without imports in the inspected file.
- `frontend/src/modules/DashboardModule/index.jsx:39-57` fetches quote summary data.
- `frontend/src/modules/DashboardModule/index.jsx:185-191` renders recent quotes.
- `backend/src/models/utils/index.js:4-49` discovers routes only from actual app model files; inspected app models include Client, Invoice, and Payment, not Quote.

**Senior Implementation Plan:**
1. Decide with product whether quote is in scope now. For a junior-safe ticket, hide broken quote UI rather than rebuilding quote.
2. Remove or comment the quote navigation item from `NavigationContainer.jsx`.
3. Remove or guard quote route entries in `routes.jsx`.
4. In `DashboardModule`, remove quote summary fetch and recent quote table, or add a feature flag-style constant that disables quote sections.
5. Ensure layout still looks balanced after removing quote dashboard cards.
6. Do not delete quote module files unless you have confirmed they are unused and product agrees.

**Manual QA:**
- Load the app after login.
- Confirm sidebar has no broken quote link.
- Confirm dashboard does not show quote cards or recent quote errors.
- Confirm invoice/payment/customer routes still work.

**Mentor Notes:** This is a reliability ticket. It is better to ship fewer visible features than visible dead ends.

## Story 11: Fix Backend Setup Missing PaymentMode And Taxes Models

**Product Story:** As a developer setting up the project, I want `npm run setup` to complete successfully so that I can start working without manual surgery.

**Why This Matters:** Setup reliability is product velocity. A junior engineer should be able to clone, install, seed, and run.

**Acceptance Criteria:**
- `npm run setup` no longer fails because of missing model files.
- Default tax and payment mode behavior is either restored or removed consistently.
- Any frontend references to disabled entities are handled.

**Code Tour:**
- `backend/src/setup/setup.js:54-67` requires `PaymentMode` and `Taxes` models and inserts default records.
- `backend/src/setup/reset.js:11-17` also references those missing models.
- `frontend/src/forms/PaymentForm.jsx:54-70` references `entity={'paymentMode'}`.
- `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:249-259` references `entity={'taxes'}`.
- `frontend/src/router/routes.jsx:110-115` references `PaymentMode` and `Taxes` route components without valid imports in the inspected file.

**Senior Implementation Plan:**
1. Choose one strategy:
   - Restore `PaymentMode.js` and `Taxes.js` models plus route pages.
   - Or remove/disable those setup and frontend dependencies until implemented.
2. For a junior engineer, prefer the smallest coherent restoration:
   - Add `backend/src/models/appModels/PaymentMode.js` with fields used by setup and form: `name`, `description`, `isDefault`, `removed`, `enabled`.
   - Add `backend/src/models/appModels/Taxes.js` with fields used by setup/form: `taxName`, `taxValue`, `isDefault`, `removed`, `enabled`.
3. Because app routes are generated from model files, adding model files should create `/api/paymentmode/...` or `/api/taxes/...` based on filename lowercasing. Check naming carefully: frontend uses `paymentMode`, but model discovery lowercases filename into `paymentmode`. This mismatch must be resolved before shipping.
4. If restoring routes is too large, change setup to skip missing models and change forms to use fallback static values.
5. Document the decision in the new model or setup comments.

**Manual QA:**
- Run `npm run setup` on a clean local database.
- Start backend.
- Confirm invoice create form can load tax options or has a safe fallback.
- Confirm payment form can load payment mode options or has a safe fallback.

**Mentor Notes:** This is harder than it looks because naming conventions matter. Dynamic route generation can save code, but it punishes inconsistent casing.

## Story 12: Add Payment Method Persistence

**Product Story:** As an admin, I want payment records to store the payment method so that I can reconcile cash, wire, and other payment types.

**Why This Matters:** The payment form already asks for payment mode, and the payment list already expects `paymentMode.name`, but the current payment schema does not define a `paymentMode` field.

**Acceptance Criteria:**
- Payment method is persisted on payment records.
- Payment list displays payment method when present.
- Recording a payment still updates invoice credit and payment status.
- Missing payment method is handled safely for old records.

**Code Tour:**
- `frontend/src/forms/PaymentForm.jsx:54-70` collects `paymentMode`.
- `frontend/src/pages/Payment/index.jsx:60-62` displays `['paymentMode', 'name']`.
- `backend/src/models/appModels/Payment.js:3-57` defines payment fields but lacks `paymentMode`.
- `backend/src/controllers/appControllers/paymentController/create.js:39-53` creates a payment from `req.body`.
- `backend/src/controllers/appControllers/paymentController/update.js:52` references `paymentMode` in update logic.

**Senior Implementation Plan:**
1. Resolve Story 11 first or decide what shape `paymentMode` has.
2. Update `Payment.js` schema:
   - If using a model reference, add `paymentMode: { type: mongoose.Schema.ObjectId, ref: 'PaymentMode', autopopulate: true }`.
   - If using a simple string, update frontend table to display the string instead of nested `.name`.
3. Align `PaymentForm` output with the schema.
4. Confirm `paymentController.create` does not strip unknown fields. It currently uses `Model.create(req.body)`, so schema support is the missing piece.
5. Update payment list rendering defensively for old records.
6. Test record payment and payment list.

**Manual QA:**
- Record a payment with default payment mode.
- Open `/payment`.
- Confirm method appears.
- Confirm invoice credit/payment status still changes.

**Mentor Notes:** This ticket teaches schema/UI alignment. If frontend expects nested `paymentMode.name`, backend must return a populated object, not a bare string.

## Story 13: Add Overdue Badge To Invoice List

**Product Story:** As an admin, I want overdue invoices clearly marked so that I can prioritize collections.

**Why This Matters:** Users should not manually compare dates and payment status to know what needs follow-up.

**Acceptance Criteria:**
- Invoice list shows an overdue badge for invoices past `expiredDate` and not paid.
- Paid invoices are not marked overdue.
- The logic is consistent after recording payments.

**Code Tour:**
- `backend/src/models/appModels/Invoice.js:132-140` defines `paymentStatus` and `isOverdue`.
- `frontend/src/pages/Invoice/index.jsx:31-43` displays invoice date and expired date.
- `frontend/src/pages/Invoice/index.jsx:74-81` displays status and payment status.
- `backend/src/controllers/appControllers/paymentController/create.js:59-77` updates `paymentStatus` after payment.

**Senior Implementation Plan:**
1. Start with frontend-only display if product accepts derived visual status:
   - In `Invoice/index.jsx`, compute overdue from `expiredDate`, `paymentStatus`, and current date.
   - Add a column or augment existing status column with an overdue tag.
2. Use `dayjs` because the file already imports it.
3. Do not mutate invoice records in the UI.
4. If product requires persisted `isOverdue`, plan a backend follow-up:
   - Update invoice read/list controllers to compute or refresh overdue.
   - Update payment create to clear overdue when fully paid.
5. Keep date boundary simple and documented: overdue means expired date is before today and paymentStatus is not `paid`.

**Manual QA:**
- Use an invoice with an expired date and unpaid status.
- Use an invoice with expired date and paid status.
- Use a future invoice.
- Confirm badges match expectations.

**Mentor Notes:** Ask whether the field should be persisted. Derived visual indicators are faster, but persisted flags can drift unless every mutation path updates them.

## Story 14: Add Invoice Number Search

**Product Story:** As an admin, I want to search invoices by invoice number so that I can find a document quickly during customer calls.

**Why This Matters:** Customer support often starts with an invoice number, not a client name.

**Acceptance Criteria:**
- The invoice list supports searching or filtering by invoice number.
- Existing client filter still works.
- Backend query input is validated or constrained.

**Code Tour:**
- `frontend/src/pages/Invoice/index.jsx:15-20` configures invoice search around client name.
- `frontend/src/modules/ErpPanelModule/DataTable.jsx:170-172` filters list by `searchConfig?.entity`.
- `frontend/src/redux/erp/actions.js:36-68` dispatches list requests with options.
- `frontend/src/request/request.js:155-174` builds list query strings.
- `backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-67` handles invoice listing.

**Senior Implementation Plan:**
1. Decide UI shape:
   - Add a text input for invoice number alongside client auto-complete.
   - Or replace client-only filter with a combined search control.
2. Update `DataTable` carefully because it is reused by invoice/payment modules.
3. If this is invoice-specific, avoid making generic `DataTable` messy. Consider passing an optional `extraFilters` render prop or adding logic behind config.
4. On backend, avoid free-form dynamic query keys. Add an explicit `number` query parameter in invoice list.
5. Parse invoice number as a number. Reject invalid values with `400`.
6. Keep client filter behavior intact by combining conditions only when both filters are present.

**Manual QA:**
- Search by existing invoice number.
- Clear search and confirm full list returns.
- Filter by client as before.
- Try invalid input and confirm no server error.

**Mentor Notes:** This is your first real "small product feature, cross-cutting implementation" story. The hardest part is not the input; it is preserving existing filter behavior.

## Story 15: Add Safer Invoice List Filtering

**Product Story:** As a system owner, I want invoice list filters to be constrained so that bad query parameters cannot create unsafe database queries.

**Why This Matters:** The invoice list controller currently uses a user-provided `filter` as a Mongo query key. That is risky.

**Acceptance Criteria:**
- Invoice list only accepts known filter fields.
- Invalid filters return a `400` response.
- Existing client filtering still works.
- Generic CRUD list behavior is not broken.

**Code Tour:**
- `backend/src/controllers/appControllers/invoiceController/paginatedList.js:10-41` reads `filter` and `equal`, then uses `[filter]: equal`.
- `backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:18-29` has a safer object-value guard for generic filters.
- `frontend/src/modules/ErpPanelModule/DataTable.jsx:170-172` sends `filter` and `equal` options based on search config.
- `frontend/src/pages/Invoice/index.jsx:15-20` sets the client filter config.

**Senior Implementation Plan:**
1. In `invoiceController/paginatedList.js`, create an allowlist such as `['client', 'createdBy', 'paymentStatus', 'status']`.
2. Build `filterCondition = {}` by default.
3. If `filter` exists:
   - Reject it if not in the allowlist.
   - Reject `equal` if it is an object.
   - Otherwise set `filterCondition = { [filter]: equal }`.
4. Replace direct `[filter]: equal` in both `Model.find` and `countDocuments` with `...filterCondition`.
5. Keep the existing fields search logic unchanged unless you are intentionally fixing search too.

**Manual QA:**
- Filter invoice list by client from UI.
- Call an invalid filter manually and confirm `400`.
- Load invoice list without filters.

**Mentor Notes:** This is a security-hardening ticket. A junior engineer can do it safely if they keep the change narrow and compare to the generic controller pattern.

## Story 16: Cap Backend Pagination Size

**Product Story:** As a system owner, I want list endpoints to cap page size so that one request cannot overload the database.

**Why This Matters:** Business tables should be fast and predictable. Unbounded `items` query params can make large database reads.

**Acceptance Criteria:**
- Generic paginated list caps `items` to a safe max.
- Invoice paginated list also caps `items`.
- Normal UI pagination still works.
- Invalid page values do not crash the backend.

**Code Tour:**
- `backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:1-7` parses page and limit.
- `backend/src/controllers/appControllers/invoiceController/paginatedList.js:5-10` has similar pagination parsing.
- `frontend/src/modules/ErpPanelModule/DataTable.jsx:153-156` sends page and page size from Ant Design table.
- `frontend/src/components/DataTable/DataTable.jsx:155-158` has a similar generic CRUD table load path.

**Senior Implementation Plan:**
1. Decide a max page size, such as 100.
2. Update generic `paginatedList.js`:
   - Parse page as integer.
   - Default page to 1.
   - Ensure page is at least 1.
   - Parse requested limit.
   - Clamp limit between 1 and 100.
3. Apply the same logic to invoice-specific `paginatedList.js`.
4. Avoid changing frontend pagination unless necessary.
5. Confirm response `pagination` still contains the current page and count.

**Manual QA:**
- Load invoice list normally.
- Load customer list normally.
- Manually request `items=100000` and confirm only capped results return.
- Manually request invalid page values and confirm stable behavior.

**Mentor Notes:** Performance hardening often looks boring. That is good. Small guardrails prevent dramatic failures.

## Story 17: Sanitize Production Error Responses

**Product Story:** As a system owner, I want production API errors to avoid exposing internal details so that users and attackers do not see stack or implementation data.

**Why This Matters:** The production error handler currently returns the raw error object even though the comment says stack traces should not leak.

**Acceptance Criteria:**
- Production error responses return a generic message.
- Development/debug details are not exposed by production handler.
- Existing validation error responses still return useful messages where intentionally handled.

**Code Tour:**
- `backend/src/handlers/errorHandlers.js:9-32` defines `catchErrors`.
- `backend/src/handlers/errorHandlers.js:39-44` defines 404 responses.
- `backend/src/handlers/errorHandlers.js:66-77` defines `productionErrors` and currently returns `error`.
- `backend/src/app.js:45-49` mounts not found and production errors.

**Senior Implementation Plan:**
1. Open `backend/src/handlers/errorHandlers.js`.
2. Modify only `productionErrors`.
3. Return:
   - `success: false`
   - `message: 'Internal server error'`
4. Do not return the raw `error` object.
5. Consider preserving `error.status` if the system later uses custom HTTP errors, but keep this first change minimal.
6. Confirm `catchErrors` still handles Joi/Mongoose validation with specific messages before production handler is reached.

**Manual QA:**
- Trigger a known bad route and confirm 404 response still works.
- Trigger a controlled server error if possible and confirm no raw stack/object is returned.
- Confirm normal invoice list still works.

**Mentor Notes:** Security work should avoid unnecessary behavior churn. Change the leak, not the whole error system.

## Story 18: Add Dashboard "Recent Payments"

**Product Story:** As an admin, I want to see recent payments on the dashboard so that I can quickly confirm incoming money.

**Why This Matters:** The dashboard already shows recent invoices and quote placeholders. Recent payments are more useful than broken quote areas in the inspected codebase.

**Acceptance Criteria:**
- Dashboard displays a recent payments table.
- It uses existing payment columns or a focused subset.
- It does not depend on missing quote backend routes.
- Existing invoice dashboard data still loads.

**Code Tour:**
- `frontend/src/modules/DashboardModule/index.jsx:127-195` renders dashboard summary cards and recent tables.
- `frontend/src/modules/DashboardModule/index.jsx:174-191` renders recent invoices and recent quotes.
- `frontend/src/pages/Payment/index.jsx:19-63` defines payment columns.
- `frontend/src/modules/DashboardModule/components/RecentTable/index.jsx` should be inspected to see how it fetches recent entity data.
- `backend/src/routes/appRoutes/appApi.js:14-17` registers entity list routes used by recent tables.

**Senior Implementation Plan:**
1. Open `RecentTable` component and understand its props.
2. Create a local `paymentDataTableColumns` array in `DashboardModule` or extract reusable payment columns carefully.
3. Replace the recent quote panel or add a new panel for recent payments.
4. Use `entity={'payment'}`.
5. Choose columns that fit dashboard width: payment number, client, amount, date.
6. If quote dashboard calls are failing due to missing backend support, remove or guard quote fetches as part of the same reliability improvement.

**Manual QA:**
- Load dashboard.
- Confirm recent payments render.
- Confirm no quote-related request errors appear if quote sections are removed.
- Confirm invoice recent table still works.

**Mentor Notes:** Dashboard work is a good lesson in product value. Do not preserve a broken placeholder when a useful existing entity can take its place.

## Story 19: Add Backend Response With Updated Invoice After Payment

**Product Story:** As an admin recording a payment, I want the frontend to receive the updated invoice so that the UI can show the latest balance immediately.

**Why This Matters:** `paymentController.create` updates the invoice but returns the payment document. The frontend then has to refetch to know the invoice state.

**Acceptance Criteria:**
- Payment create response includes the created payment and updated invoice.
- Existing frontend payment success flow still works.
- The invoice list refresh still occurs.
- No payment credit/status logic changes.

**Code Tour:**
- `backend/src/controllers/appControllers/paymentController/create.js:41-53` creates and updates the payment PDF path.
- `backend/src/controllers/appControllers/paymentController/create.js:56-77` updates invoice credit and payment status.
- `backend/src/controllers/appControllers/paymentController/create.js:79-83` currently returns `updatePath`.
- `frontend/src/redux/erp/actions.js:98-118` handles `recordPayment` success.
- `frontend/src/modules/InvoiceModule/RecordPaymentModule/components/RecordPayment.jsx:33-40` resets and navigates after success.

**Senior Implementation Plan:**
1. In `paymentController/create.js`, keep the existing payment creation and invoice update logic.
2. Change the response `result` shape from just payment to an object:
   - `payment: updatePath`
   - `invoice: invoiceUpdate`
3. Update `frontend/src/redux/erp/actions.js` carefully:
   - It currently expects `data.result.invoice`.
   - This may actually become more correct if response includes `invoice`.
4. Verify any other payment create callers expect the old shape.
5. Update frontend summary story if you use this result for balance display.

**Manual QA:**
- Record a payment.
- Confirm API response includes both payment and invoice.
- Confirm frontend still navigates back to invoice list.
- Confirm invoice list shows updated credit/payment status.

**Mentor Notes:** Response-shape changes are contract changes. Always search for all consumers before changing the backend.

## Story 20: Add A Lightweight Backend Test Harness For Payment Rules

**Product Story:** As a product owner, I want payment rules protected by tests so that future changes do not break invoice balances.

**Why This Matters:** Payment creation changes both a payment record and an invoice record. That is high-risk business logic.

**Acceptance Criteria:**
- Backend has a test script in `backend/package.json`.
- At least one test covers partial payment updating invoice credit and payment status.
- The test setup is documented in the test file or nearby docs.
- Production code is not modified just to make the test pass.

**Code Tour:**
- `backend/package.json:8-15` currently has no test script.
- `backend/src/controllers/appControllers/paymentController/create.js:9-86` contains the business logic to test.
- `backend/src/models/appModels/Invoice.js:118-136` defines credit and payment status.
- `backend/src/models/appModels/Payment.js:20-39` defines invoice, amount, and currency fields.
- `backend/src/server.js:13-39` connects MongoDB globally for runtime, so tests should avoid accidentally starting the server.

**Senior Implementation Plan:**
1. Pick a test runner. For this repo, Jest or Vitest would both work, but choose one and add it as a dev dependency.
2. Add a backend `test` script. Do not modify frontend scripts.
3. Create a test file near the payment controller or in a backend test folder.
4. Use a test database strategy:
   - Preferred: in-memory MongoDB if adding dependency is acceptable.
   - Simpler local option: require `DATABASE_TEST` and skip destructive behavior against production DB.
5. Register Mongoose models in the test before calling controller logic.
6. Create fixtures:
   - Admin
   - Client
   - Invoice with total 100, discount 0, credit 0, paymentStatus `unpaid`
7. Mock Express `req` and `res`.
8. Call `paymentController.create`.
9. Assert:
   - Response status is 200.
   - Payment was created.
   - Invoice credit increased.
   - Invoice paymentStatus is `partially`.
10. Add one negative test later for amount greater than max.

**Manual QA:**
- Run backend tests locally.
- Run backend dev server afterward to ensure test setup did not change runtime behavior.
- Record a payment manually if a local seeded database is available.

**Mentor Notes:** This is the capstone junior story. The goal is not perfect test architecture on day one. The goal is to protect the most valuable business rule and learn how to test controller behavior without launching the whole app.

