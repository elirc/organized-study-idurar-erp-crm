# Verification Log

Running record of what was inspected while writing this curriculum, kept as work proceeded (not reconstructed at the end). Date: 2026-07-09. Environment: Windows 11, repo at `masteridur/idurar-erp-crm`, git branch `main` with **no commits yet** (`git log` → "does not have any commits yet"). No MongoDB instance available, so nothing was executed against a live DB — all runtime claims are derived from reading source, and every command in the cheatsheet is marked **inferred** unless noted.

## Method

- File inventory via directory listing of `backend/src` (116 files) and `frontend/src` (258 files).
- Read every file cited in an anchor immediately before writing that anchor; line numbers confirmed against the file at read time.
- Where behavior depends on runtime data (DB state, env vars), the claim is labeled or traced through code rather than asserted.

## Backend files inspected (read in full unless noted)

| File | Purpose confirmed | Notes |
| --- | --- | --- |
| `backend/src/server.js` | boot, model glob, mongoose connect | Node<20 guard at :7-11 |
| `backend/src/app.js` | router topology, auth placement | `/download` `/public` unauthenticated (:42-43) |
| `backend/src/routes/appRoutes/appApi.js` | generated per-entity routes | invoice/quote/payment special-cased (:19-25) |
| `backend/src/routes/coreRoutes/coreAuth.js` | login/logout/forget/reset | logout behind auth (:13) |
| `backend/src/routes/coreRoutes/coreApi.js` | admin + settings routes | |
| `backend/src/routes/coreRoutes/coreDownloadRouter.js` | PDF download by filename | no auth, no `removed` filter downstream |
| `backend/src/routes/coreRoutes/corePublicRouter.js` | static file serve | path-traversal guard (:24-29) |
| `backend/src/controllers/middlewaresControllers/createAuthMiddleware/*` | login, authUser, isValidAuthToken, logout | dual-token session confirmed |
| `backend/src/controllers/middlewaresControllers/createCRUDController/*` | generic CRUD | `paginatedList.js:19-29` has object-guard for injection |
| `backend/src/controllers/appControllers/index.js` | custom-vs-generic controller resolution | |
| `backend/src/controllers/appControllers/invoiceController/*` | create/update/read/remove/summary/paginatedList/sendMail/schemaValidate | `sendMail` is a premium stub (:5-11) |
| `backend/src/controllers/appControllers/paymentController/*` | create/update/remove/summary | three-write no-transaction sequence confirmed |
| `backend/src/controllers/appControllers/clientController/summary.js` | `$facet`+`$lookup` aggregation | |
| `backend/src/controllers/pdfController/index.js` | Pug → html-pdf | regenerates each request (:25-27) |
| `backend/src/handlers/errorHandlers.js` | `catchErrors`, 404, prod errors | echoes `error` to client (:22-28) |
| `backend/src/handlers/downloadHandler/downloadPdf.js` | download orchestration | `findOne` without `removed:false` (:9-11) |
| `backend/src/models/appModels/{Invoice,Payment,Client}.js` | schemas | autopopulate plugin on all three |
| `backend/src/models/coreModels/{Admin,AdminPassword,Setting}.js` | schemas | `loggedSessions` array (`AdminPassword.js:30-33`) |
| `backend/src/models/utils/index.js` | filename→route derivation | |
| `backend/src/helpers.js` | `calculate` money helpers | |
| `backend/src/middlewares/settings/increaseBySettingKey.js` | atomic `$inc`, swallows errors | (:29-31) |
| `backend/src/middlewares/uploadMiddleware/singleStorageUpload.js` | multer disk storage | |
| `backend/src/setup/setup.js` | seed admin/settings/taxes/paymentmode | **requires missing models** (:54-67) |
| `backend/src/utils/is-path-inside.js` | containment check | local reimplementation of npm pkg |

## Frontend files inspected

| File | Purpose confirmed | Notes |
| --- | --- | --- |
| `frontend/src/main.jsx`, `RootApp.jsx`, `apps/IdurarOs.jsx`, `apps/ErpApp.jsx` | bootstrap + auth gate + settings load | `settingsAction.list` at `ErpApp.jsx:36-38`, render gated on `isSuccess` (:42) |
| `frontend/src/router/{routes.jsx,AppRouter.jsx,AuthRouter.jsx}` | route table | `routes.jsx:73-116` references removed `Quote`/`PaymentMode`/`Taxes` — **broken** |
| `frontend/src/request/{request.js,errorHandler.js}` | axios wrapper + envelope | errors coerced to `{success:false}` |
| `frontend/src/auth/auth.service.js` | auth API calls | |
| `frontend/src/redux/erp/{actions,reducer,selectors}.js` | generic entity store | `keyState` pattern |
| `frontend/src/redux/auth/actions.js` | auth thunks, localStorage writes | |
| `frontend/src/redux/{store,storePersist}.js` | store config, localStorage persistence | |
| `frontend/src/modules/ErpPanelModule/{index,DataTable,CreateItem}.jsx` | config-driven CRUD UI | invoice special-casing in DataTable |
| `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx` | invoice form | number seeded from `last_invoice_number+1` (:40) |
| `frontend/src/modules/InvoiceModule/RecordPaymentModule/*` | payment recording | client `maxAmount` guard |
| `frontend/src/pages/Invoice/index.jsx` | invoice page config | column definitions |
| `frontend/src/settings/useMoney.jsx`, `frontend/src/utils/calculate.js` | money formatting/math | duplicated with backend equivalents |
| `frontend/src/config/serverApiConfig.js` | API base URLs | localhost:8888 in dev |
| `frontend/vite.config.js` | dev server + proxy | port 3000, proxies `/api` |
| `frontend/src/hooks/*` | useFetch/useMail/useDebounce/etc | |

## Confirmed defects of this copy (not just suspicions)

1. **Setup crashes** — `setup.js:54-67` requires `../models/appModels/PaymentMode` and `Taxes`, absent from `models/appModels/` (only Client/Invoice/Payment present). Confirmed by directory listing. Admin+settings seed before the throw.
2. **Broken frontend routes** — `routes.jsx` renders `<Quote/>`, `<QuoteCreate/>`, `<QuoteRead/>`, `<QuoteUpdate/>`, `<PaymentMode/>`, `<Taxes/>` (`:73-116`) with no corresponding lazy imports (only imports at `:5-26` exist). Will throw `ReferenceError` at module eval. Nav links to `/quote`, `/payment/mode`, `/taxes` also dangle.
3. **Download route unauthenticated + ignores soft delete** — `app.js:42` + `downloadPdf.js:9-11`.
4. **Invoice summary ignores its date range** — `invoiceController/summary.js:27-29` computes `startDate`/`endDate` but the `$match` at `:33-42` has the date filter commented out, so `type=week|month|year` doesn't scope the total. Confirmed by reading the whole file.
5. **`loggedSessions` unbounded growth** — `authUser.js:23-29` `$push` with no prune; `AdminPassword.js:30-33`.

## Uncertainties / not fully verified (labeled in-text as *investigate*)

- Exact runtime status codes were traced through code, not observed against a running server.
- ReDoS feasibility of the `new RegExp(req.query.q)` in `paginatedList.js:14-16` / `search.js:17-19` is plausible but not demonstrated with a payload.
- Whether `npm run lint` passes clean was not executed.

## Areas not covered in depth

- Localization/translation files (`locale/translation/*`) beyond the `useLanguage` mechanism.
- The 40+ marketing README translations under `features/`.
- Email templates (`emailTemplate/`) and the Resend/OpenAI integrations (present in deps, lightly wired).
- Inventory middleware (`middlewares/inventory/`) — appears vestigial; not traced.

## Re-check on 2026-10-07 (study copy `elirc/organized-study-idurar-erp-crm`)

Static re-check of `fable docs/` and `docs2/`; nothing installed or run.

- **Defects 1–5 re-confirmed at the same lines.**
  1. `setup.js` L54–55 requires `PaymentMode`/`Taxes`; `models/appModels/` still holds only `Client.js`, `Invoice.js` and `Payment.js`.
  2. `routes.jsx` L73–86 renders `<Quote/>`, `<QuoteCreate/>`, `<QuoteRead/>` and `<QuoteUpdate/>`.
  3. `app.js` L42–43 mounts `/download` and `/public`; `downloadPdf.js` L9–11 calls `findOne({ _id: id })` without `removed`.
  4. `invoiceController/summary.js` L37–40 has the date `$match` commented out.
  5. `authUser.js` L25 runs `$push: { loggedSessions: token }`.
- **Anchors:** every line-anchored citation in both trees lands inside its file. Missing paths are either proposals (`backend/jest.config.js`, `backend/test/*`, `.github/workflows/ci.yml`, the `Widget.js` drill, the `PaymentMode.js`/`Taxes.js` restorations in `docs2/`) or the git-ignored `backend/.env`.
- **Drift fixed:** `04-runtime-and-tooling-map.md` said there was no `.github/workflows`. The folder exists, with `codesee-arch-diagram.yml` (CodeSee diagram action, `continue-on-error: true`), `github-repo-stats.yml` (daily stats cron) and a misplaced `CODEOWNERS`. None of them builds or tests, so the curriculum's "no CI" remains true in the sense of no build/test pipeline.
