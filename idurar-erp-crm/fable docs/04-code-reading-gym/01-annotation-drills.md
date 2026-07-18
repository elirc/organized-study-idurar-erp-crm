# Annotation Drills

For each excerpt, open the real file and annotate — on paper or in comments — six things: **inputs**, **outputs**, **dependencies**, **invariants** (what must be true), **side effects**, **failure modes**. Then check the rubric. A strong annotation names an invariant and a failure mode the code doesn't state.

---

## Drill 1: `paginatedList.js` (generic list)

File: `backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:1-72`.

Annotate before reading on. Then verify:
- **Inputs:** `req.query` — `page`, `items`, `sortBy`, `sortValue`, `filter`, `equal`, `fields`, `q`.
- **Outputs:** 200 `{result, pagination}` or 203 `{result:[]}` when empty.
- **Dependencies:** the Mongoose `Model`, MongoDB.
- **Invariants:** `page ≥ 1`; only non-`removed` docs returned; `equal` must be a primitive (`:21-27`).
- **Side effects:** none (read-only).
- **Failure modes:** `page='abc'` → `skip(NaN)`; regex from `q` unescaped (`:15`) → ReDoS; `countDocuments` on every call is a second scan.

Self-grade — Solid: you found the read-only nature and the `removed:false` filter. Strong: you flagged the `NaN`-skip and the unescaped regex as failure modes.

---

## Drill 2: `paymentController/create.js` (the invariant)

File: `backend/src/controllers/appControllers/paymentController/create.js:9-84`.

- **Inputs:** `req.body` (amount, invoice, client…), `req.admin._id`.
- **Outputs:** 200 with the payment, or 202 on `amount===0`/overpay.
- **Dependencies:** Payment + Invoice models, `calculate`.
- **Invariants:** `amount > 0`; `amount ≤ total - discount - credit`; after success `invoice.credit === old credit + amount` and `paymentStatus` reflects it.
- **Side effects:** creates a Payment (`:41`), updates it with a pdf name (`:43`), updates the Invoice (`:66-77`) — **three writes, no transaction**.
- **Failure modes:** crash between write #1 and #3 breaks the invariant; retry double-counts; 202 (not 400) for a validation failure.

Self-grade — Strong: you named the crash window and the retry double-count without prompting.

---

## Drill 3: `isValidAuthToken.js` (per-request auth)

File: `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:5-71`.

- **Inputs:** `Authorization` header, `JWT_SECRET`.
- **Outputs:** calls `next()` with `req.admin` set, or 401.
- **Dependencies:** `jwt`, User + UserPassword models.
- **Invariants:** token must verify AND be present in `loggedSessions` (`:45-53`).
- **Side effects:** sets `req[userModel.toLowerCase()]` (`:55-57`).
- **Failure modes:** doesn't check `enabled` (a disabled user's existing token still works); any error → 500 (not 401) with `jwtExpired:true`; the parallel fetch (`:32-35`) means both queries run even if the user is missing.

Self-grade — Strong: you spotted that `enabled` isn't re-checked here (only at login).

---

## Drill 4: `invoiceController/create.js` (business logic)

File: `backend/src/controllers/appControllers/invoiceController/create.js:9-72`.

- **Inputs:** `req.body`, `req.admin._id`.
- **Outputs:** 200 with the invoice, or 400 on Joi failure.
- **Invariants:** `total = subTotal + taxTotal`; `subTotal = Σ(qty×price)`; `createdBy` is the caller.
- **Side effects:** saves invoice, second write for pdf name (`:52-59`), un-awaited counter bump (`:62-64`).
- **Failure modes:** client `total`/`item.total` are overwritten (good); Mongoose throw after Joi pass → 500; counter bump failure is silent.

Self-grade — Solid: you saw the totals are recomputed. Strong: you noticed the counter bump isn't awaited and can't fail the request.

---

## Drill 5: `redux/erp/reducer.js` (generic reducer)

File: `frontend/src/redux/erp/reducer.js:70-127`.

- **Inputs:** `action.type`, `action.keyState`, `action.payload`.
- **Outputs:** new state object.
- **Invariants:** each `keyState` slice has `{result, isLoading, isSuccess}`; state is never mutated (spreads).
- **Side effects:** none (pure).
- **Failure modes:** a bad `keyState` writes a new key silently; `REQUEST_SUCCESS` *replaces* the slice (`:100-107`), dropping any fields not re-supplied.

Self-grade — Strong: you noticed `REQUEST_SUCCESS` overwrites rather than merges the slice.

---

## Drill 6: `AutoCompleteAsync/index.jsx` (async component)

File: `frontend/src/components/AutoCompleteAsync/index.jsx:55-112`.

- **Inputs:** props (`entity`, `searchFields`, `value`, `onChange`), user keystrokes.
- **Outputs:** a Select with async options; calls `onChange`.
- **Invariants:** searches fire only after 500ms of quiet (`:55-62`).
- **Side effects:** network request per debounced value (`:74-85`); navigates on redirect option (`:46-48`).
- **Failure modes:** no request cancellation → out-of-order results win (`useOnFetch` has no abort); two `useRef`s track edit-vs-search to avoid loops.

Self-grade — Strong: you identified the out-of-order async hazard.

---

## Drill 7: `downloadPdf.js` (public boundary)

File: `backend/src/handlers/downloadHandler/downloadPdf.js:4-74`.

- **Inputs:** `directory`, `id` parsed from the filename.
- **Outputs:** streams a PDF, or 400/404/500 JSON.
- **Invariants:** the model must exist; **note there's no `removed:false`** (`:9-11`).
- **Side effects:** generates and writes a file every call.
- **Failure modes:** no auth on this route (`app.js:42`); soft-deleted invoices still download; regeneration cost per hit.

Self-grade — Strong: you connected the missing `removed` filter and the route being unauthenticated.

---

## Drill 8: `errorHandlers.js` (`catchErrors`)

File: `backend/src/handlers/errorHandlers.js:9-32`.

- **Inputs:** an async handler `fn`.
- **Outputs:** a wrapped handler that catches rejections.
- **Invariants:** `ValidationError` → 400, else → 500.
- **Side effects:** sends a response on error.
- **Failure modes:** business failures that `return res.json` (not throw) bypass it; echoes the raw `error` to the client (`:17,26`); if `fn` already responded then throws → double-send.

Self-grade — Strong: you flagged the info-leak of the raw error object.

---

## Drill 9: `singleStorageUpload.js` (multer)

File: `backend/src/middlewares/uploadMiddleware/singleStorageUpload.js:7-56`.

- **Inputs:** `{entity, fileType, fieldName}`, the uploaded file.
- **Outputs:** middleware that stores the file and sets `req.upload` + `req.body[fieldName]`.
- **Invariants:** files go to `src/public/uploads/${entity}`; filename slugified + random suffix (`:20-30`).
- **Side effects:** writes a file to disk; mutates `req.body`.
- **Failure modes:** the field must be named `file` (`:54` hardcodes `.single('file')`); `req.body.seotitle` read before body parse ordering; no size limit visible here.

Self-grade — Strong: you noticed the hardcoded `'file'` field name vs the parameterized `fieldName`.

---

## Drill 10: `clientController/summary.js` (aggregation)

File: `backend/src/controllers/appControllers/clientController/summary.js:6-95`.

- **Inputs:** `req.query.type` (week/month/year).
- **Outputs:** 200 `{new, active}` percentages, or 400 on bad type.
- **Dependencies:** Client + Invoice collections.
- **Invariants:** `type` whitelisted (`:10-18`); divide-by-zero guarded (`:84-85`).
- **Side effects:** none.
- **Failure modes:** `$lookup` (`:52-58`) joins invoices — miss it and you'd write an N+1; percentages round, so tiny counts distort.

Self-grade — Strong: you recognized the `$lookup` as the N+1 avoidance and the input whitelist as good practice.

---

## Meta-drill

Pick any two drills and, without looking, state the *invariant* each protects. If you can't, you read for "what it does" not "what must stay true" — reread focusing on the invariant line. Invariants are what interviewers probe ("what breaks that?").
