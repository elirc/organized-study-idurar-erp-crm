# Framework Mental Models

One mental model per framework, then where IDURAR uses it well and badly.

## Express 4 — middleware pipeline

**Mental model:** a request flows through an ordered list of `(req, res, next)` functions. Each either responds or calls `next()`. Order is everything; the first to respond wins.

**In this repo:** `app.js:21-46` is the pipeline — cors → cookieParser → json/urlencoded → compression → route mounts → 404 → error handler. Auth is *positional middleware*: `app.use('/api', adminAuth.isValidAuthToken, coreApiRouter)` (`:40`) inserts the auth check before the router. The routes that skip it (`/api` auth routes at `:39`, `/download` at `:42`, `/public` at `:43`) simply omit it from their `use` call.

**Well:** the `catchErrors` wrapper (`handlers/errorHandlers.js:9-32`) adapts async handlers to Express 4's non-awaiting model — a clean solution to a real framework limitation.

**Sharp edge:** because auth is positional, forgetting to include it on a new mount silently makes it public (exactly what happened to `/download`). There's no "secure by default"; security is an ordering discipline.

**Pitfall checklist:** middleware order matters; `express.json()` must precede handlers that read `req.body`; error middleware must have 4 args to be recognized; a handler that responds and then calls `next()` double-sends.

**Interview angle:** Q1 ([08/01](../08-interview-prep/01-js-ts-node-deep-dive.md)) and Q4 ([08/03](../08-interview-prep/03-api-and-data-modeling-questions.md)).

## Mongoose 8 — schema, model, document

**Mental model:** a **schema** defines shape + validation; a **model** is the collection interface (`Model.find`, `Model.create`); a **document** is one instance with methods and hooks. Refs + `populate` simulate joins.

**In this repo:** schemas in `models/appModels/`. `Invoice.js` shows the full toolkit — types, `required`, `enum` (`status:148-152`), `default`, embedded arrays (`items:51-95`), refs (`client:31-36`), and the `mongoose-autopopulate` plugin (`:178`) that auto-joins refs on every query.

**Well:** `runValidators:true` on updates (`createCRUDController/update.js:12`) keeps schema constraints enforced on edits, not just creates.

**Sharp edge:** global `autopopulate` causes invisible over-fetching and cascading joins (`Payment` → `invoice` → `client`), a hidden N+1 on lists. Explicit `.populate('createdBy','name')` (`invoiceController/read.js:11`) is the disciplined alternative but is used inconsistently.

**Pitfall checklist:** `findOneAndUpdate` skips `save` hooks and defaults unless flagged; instance methods must be `function`s (for `this`); a missing `removed:false` filter leaks soft-deleted docs; blindly spreading `req.body` into an update is mass assignment (`createCRUDController/update.js:9`).

**Interview angle:** Q14 ([08/01](../08-interview-prep/01-js-ts-node-deep-dive.md)), Q3/Q6/Q13 ([08/03](../08-interview-prep/03-api-and-data-modeling-questions.md)).

## React 18 — declarative UI, render vs commit

**Mental model:** your component is a function of props+state that returns elements (render). React diffs the result and applies changes to the DOM (commit). Hooks let function components hold state (`useState`) and run side effects (`useEffect`) tied to a dependency array. You describe *what* the UI should be, not *how* to mutate it.

**In this repo:** all function components + hooks. `DataTable.jsx` reads Redux via `useSelector`, dispatches on interaction, and re-renders when the selected slice changes. `ItemRow.jsx` and `AutoCompleteAsync` show local `useState` + effects.

**Well:** config-driven components (`DataTable({config})`) — one implementation, many thin page configs (`pages/Invoice/index.jsx`).

**Sharp edge:** effect dependency arrays are error-prone. `DataTable.jsx:162-168` (`[]` reading `entity`), `AutoCompleteAsync:74-103` (chained effects with an async race), `ItemRow.jsx:9-52` (derived state duplicated in `useState`). Classic hook pitfalls, all present.

**Pitfall checklist:** missing effect deps → stale closures; storing derived data in state → drift; missing/unstable `key` in lists → wrong DOM reuse; async effects without cleanup → out-of-order results and state-after-unmount.

**Interview angle:** Q1, Q2, Q3, Q7 ([08/02](../08-interview-prep/02-frontend-framework-questions.md)).

## Redux Toolkit — single store, actions, reducers, selectors

**Mental model:** one immutable state tree; the only way to change it is dispatching an action to a reducer that returns new state; components read via selectors and re-render when their selected slice changes. Thunks handle async.

**In this repo:** a **generic** entity store — one reducer keyed by `keyState` handles list/create/read/update/delete for every entity (`redux/erp/reducer.js:70-127`); thunks hand-roll the loading→success/failed lifecycle (`redux/erp/actions.js:36-68`); memoized selectors slice reads (`selectors.js:5-22`).

**Well:** genuine reuse — invoices, payments, clients all share one reducer and one set of thunks.

**Sharp edge:** `keyState` is a magic string (typo = silent new key); server data lives in Redux as hand-managed cache (what RTK Query exists to replace); the global singleton means two entity tables can collide (worked around by `resetState` on mount, `ErpPanelModule/index.jsx:21-27`).

**Pitfall checklist:** returning a new object from a selector each call breaks memoization; factory selectors (`selectItemById`) called inline defeat caching; mutating state in a reducer (RTK's Immer allows *draft* mutation, but this repo uses spreads).

**Interview angle:** Q4, Q9, Q10 ([08/02](../08-interview-prep/02-frontend-framework-questions.md)).

## Ant Design 5 — component library + Form engine

**Mental model:** prebuilt components; the `Form` holds field state internally, keyed by `name` (including array paths for lists). You read/write via the form instance, not manual state.

**In this repo:** `Form.List name="items"` drives dynamic invoice line items (`InvoiceForm.jsx:193-212`), each `ItemRow` binding inputs with `[field.name, 'price']` array paths (`ItemRow.jsx:84`). `<Table>` is configured with `columns` + `dataSource` (`DataTable.jsx:204-213`).

**Sharp edge:** shadowing Ant's form state with local `useState` (`ItemRow.jsx:9-11`) creates two sources of truth for the same value.

**Interview angle:** Q7 ([08/02](../08-interview-prep/02-frontend-framework-questions.md)).

## Drill

For each framework, name one thing this repo does *well* and one *sharp edge*, from memory. Self-grade — Strong: you can tie each sharp edge to a specific file and explain the failure mode, not just name it.
