# Frontend / React Question Cards

12 cards on React 18 + Redux Toolkit + Ant Design as this repo uses them. Practice aloud, 90 seconds each. Framework: React 18 function components + hooks, Redux Toolkit store, react-router-dom v6, Ant Design v5 forms.

---

## Q1: Explain React's render vs commit using this table component.

Round: frontend. What it's testing: the two-phase model.

Repo anchor: `frontend/src/modules/ErpPanelModule/DataTable.jsx:204-213` (Ant `<Table>` re-renders on `dataSource`/`loading` change).

- **Junior:** "React re-renders when state changes."
- **Mid:** "Render is React calling the component to compute the next element tree — pure, no DOM touched. Commit is when it applies the diff to the DOM. `DataTable` re-renders whenever `listResult` or `listIsLoading` from `useSelector` change; only the changed table cells are committed."
- **Senior:** "The distinction matters for performance: an expensive render that produces the same output still costs CPU even if commit is cheap. That's what `React.memo`/`useMemo` address — skipping the render, not the commit."

Follow-up: "What triggers a re-render here besides state?" (New Redux slice reference from `useSelector`, parent re-render, new `config` prop identity.)

---

## Q2: This `useEffect` has an empty dependency array but reads `entity`. Bug or fine?

Round: frontend. What it's testing: stale closures, dependency arrays.

Repo anchor: `DataTable.jsx:162-168` (effect with `[]` deps calling `dispatcher` which closes over `entity`).

- **Junior:** "It runs once on mount, that's fine."
- **Mid:** "It captures `entity` at mount. Since each entity page mounts a fresh `DataTable`, `entity` never changes during the component's life, so it works — but it's *accidentally* correct. If `entity` could change without remount, the effect would use the stale value."
- **Senior:** "The lint rule `react-hooks/exhaustive-deps` would flag it. The honest fix is to include `entity` (and memoize `dispatcher`) or acknowledge the mount-only intent explicitly. The `AbortController` created here (`:163`) is also never wired to the dispatch — dead cleanup code."

Follow-up: "What's a stale closure and where else is one lurking?" (`ItemRow.jsx:46` effect on `[current]` reading `field.fieldKey`.)

---

## Q3: Walk me through the debounced async search component.

Round: frontend. What it's testing: debouncing, effect chains, race conditions.

Repo anchor: `frontend/src/components/AutoCompleteAsync/index.jsx:55-103`.

- **Junior:** "It waits before searching."
- **Mid:** "Typing sets `valToSearch` (`:92`); `useDebounce` (`:55-62`) copies it to `debouncedValue` after 500ms of quiet; an effect on `[debouncedValue]` (`:74-85`) fires the request via `onFetch`; a second effect on `[isSuccess, result]` (`:95-103`) pushes results into options. It avoids a request per keystroke."
- **Senior:** "There's an out-of-order hazard: `onFetch` (`useOnFetch.jsx:8-19`) has no request cancellation, so if an earlier slow search resolves after a later fast one, stale results win. The cleanup only cancels the debounce timer (`:82-84`), not the in-flight request. I'd use an `AbortController` or a request-id guard."

Follow-up: "Why two `useRef`s (`isUpdating`, `isSearching`)?" (To distinguish programmatic value-setting for edit forms from user searches without triggering re-renders.)

---

## Q4: How does state flow from a click to the server and back in this app?

Round: frontend. What it's testing: unidirectional data flow, Redux thunks.

Repo anchor: `redux/erp/actions.js:36-68` (thunk), `redux/erp/reducer.js:83-108` (reducer), `redux/erp/selectors.js:7` (selector), `DataTable.jsx:47` (`useSelector`).

- **Junior:** "Redux stores the data."
- **Mid:** "Click → `dispatch(erp.list({entity}))` → thunk dispatches `REQUEST_LOADING`, awaits `request.list`, dispatches `REQUEST_SUCCESS` with results → reducer writes `state.list` → `useSelector(selectListItems)` re-renders the table. One direction, always."
- **Senior:** "It's a hand-rolled version of what RTK Query does for free — loading/success/error lifecycle, cache, dedup. For a CRUD app I'd argue RTK Query or React Query eliminates most of `actions.js` and the generic reducer."

Follow-up: "Where's the loading state stored and how does the table read it?" (`state.list.isLoading` → `selectListItems` → `listIsLoading` at `DataTable.jsx:47`.)

---

## Q5: Why does the app store the JWT in localStorage and what's the risk?

Round: frontend/security. What it's testing: token storage tradeoffs.

Repo anchor: `redux/auth/actions.js:20`, `frontend/src/request/request.js:16-25`.

- **Junior:** "So it persists across refreshes."
- **Mid:** "Login writes the auth object (with token) to localStorage (`actions.js:20`); `request.js:16-25` reads it and sets the `Authorization: Bearer` header on axios. Persists across reloads without a server round-trip."
- **Senior:** "localStorage is readable by any JavaScript, so a single XSS exfiltrates the token — no `HttpOnly` protection. The commented-out cookie code in `authUser.js:31-39` shows they considered `httpOnly` cookies, which resist XSS but need CSRF protection. It's the classic localStorage-vs-cookie tradeoff and they picked the more XSS-exposed option."

Follow-up: "What would make XSS less catastrophic here?" (Short token TTL, `httpOnly` cookie + CSRF token, strict CSP.)

---

## Q6: Explain code splitting in this app and one way it's currently broken.

Round: frontend. What it's testing: lazy loading, Suspense, error boundaries.

Repo anchor: `frontend/src/router/routes.jsx:5-26` (lazy) and `:73-116` (references to un-imported components).

- **Junior:** "Lazy loads pages."
- **Mid:** "Each route is `lazy(() => import(...))` wrapped in `<Suspense>` (`IdurarOs.jsx`, `RootApp.jsx`), so the login bundle doesn't include the whole app. Chunks load on navigation."
- **Senior:** "It's broken in this copy: `routes.jsx:73-116` renders `<Quote/>`, `<PaymentMode/>`, `<Taxes/>` but their lazy imports were deleted — evaluating the module throws `ReferenceError`. And there's no error boundary, so a failed chunk load (network blip) would crash the app instead of showing a retry. I'd add an `ErrorBoundary` around the router."

Follow-up: "What does the user see during a slow chunk load?" (The `<PageLoader/>` Suspense fallback.)

---

## Q7: Ant Design `Form.List` powers invoice line items. How does it work here?

Round: frontend. What it's testing: controlled dynamic forms, keys.

Repo anchor: `frontend/src/modules/InvoiceModule/Forms/InvoiceForm.jsx:193-212` and `ItemRow.jsx:8-52`.

- **Junior:** "It's a list of inputs."
- **Mid:** "`Form.List name=\"items\"` (`InvoiceForm.jsx:193`) gives `fields` plus `add`/`remove`. Each `ItemRow` binds its inputs with array-path names like `[field.name, 'price']` (`ItemRow.jsx:84`), so Ant manages the array in form state. `key={field.key}` keeps React reconciliation stable across add/remove (`:196-197`)."
- **Senior:** "There's local `useState` for `price`/`quantity`/`total` *duplicating* what the form already holds (`ItemRow.jsx:9-11`), plus an effect recomputing total (`:48-52`). That's derived state stored twice — a common React smell. The row total should be computed from form values, not shadowed in `useState`."

Follow-up: "Why is `key` critical when removing a middle row?" (Without stable keys, React reuses the wrong DOM node and inputs show the wrong values.)

---

## Q8: The auth gate renders one of two apps. Walk me through it.

Round: frontend. What it's testing: conditional rendering, route guarding.

Repo anchor: `frontend/src/apps/IdurarOs.jsx:60-71` (`isLoggedIn ? DefaultApp : AuthRouter`).

- **Junior:** "It checks if you're logged in."
- **Mid:** "`useSelector(selectAuth)` gives `isLoggedIn`; if false it renders `<AuthRouter>` (login/reset routes only), if true the lazy `<ErpApp>` with the full route table. The auth state is seeded from localStorage at store creation (`redux/store.js`), so a refresh keeps you logged in."
- **Senior:** "This is client-side-only guarding — it hides UI but the real protection is the server's `isValidAuthToken` on every `/api` call (`app.js:40-41`). A junior mistake would be to think the `isLoggedIn` check *is* the security. It isn't; it's UX. The server never trusts it."

Follow-up: "What happens if the token is expired but localStorage still says logged-in?" (UI renders, first API call returns 401 with `jwtExpired:true`, handled by the request error layer.)

---

## Q9: `useSelector` with `createSelector` — why the memoization?

Round: frontend. What it's testing: selector memoization, re-render control.

Repo anchor: `frontend/src/redux/erp/selectors.js:5-22`.

- **Junior:** "It selects from the store."
- **Mid:** "`createSelector` (reselect) memoizes: `selectListItems` only recomputes if `state.erp.list` changed by reference. Without it, a selector that builds a new object each call would return a new reference every render and force re-renders even when data didn't change."
- **Senior:** "`selectItemById` (`selectors.js:8-9`) is a *factory* selector — it takes an id and returns a new selector. Called inline in render it creates a fresh selector each time, defeating memoization. For per-item selection you'd `useMemo` the selector or use RTK entity adapters."

Follow-up: "When does `useSelector` trigger a re-render?" (When the selected value fails a strict `===` equality vs the previous.)

---

## Q10: Where should state live — this app puts everything in Redux. Right call?

Round: frontend. What it's testing: state placement, local vs global.

Repo anchor: global: `redux/erp/reducer.js`; local: `ItemRow.jsx:9-11`, `AutoCompleteAsync/index.jsx:26-35`.

- **Junior:** "Redux is for state."
- **Mid:** "Server data (lists, current item) is global in Redux because many components need it. But ephemeral UI state — a search box's text (`AutoCompleteAsync:34`), a row's local total (`ItemRow:9`) — is correctly kept in `useState`, not Redux. The rule: server cache and cross-cutting state global; transient UI state local."
- **Senior:** "The mistake here is the *reverse* — server data in Redux is really *cache*, and hand-managing cache in Redux (`erp/reducer.js`) is what RTK Query/React Query exist to replace. I'd move server state to a query cache and keep Redux for genuine client state (auth, settings)."

Follow-up: "Name a bug caused by putting server data in a global singleton." (Two entity tables share `state.erp`; the `resetState` on mount at `ErpPanelModule/index.jsx:21-27` is the workaround for exactly that.)

---

## Q11: How is i18n / translation implemented, and what's the tradeoff?

Round: frontend. What it's testing: pragmatic i18n, lazy label generation.

Repo anchor: `frontend/src/locale/useLanguage.jsx` (auto-generates a label from the key), used as `translate('add_new_invoice')` throughout.

- **Junior:** "It translates text."
- **Mid:** "`useLanguage` returns a `translate(key)` that looks up a language file and, on a miss, *derives* a human label from the key itself (title-casing `add_new_invoice` → 'Add New Invoice') and caches it in localStorage. So missing translations degrade to readable English instead of showing raw keys."
- **Senior:** "Clever for coverage but it means you never *know* what's untranslated — every key silently 'works'. A stricter setup (i18next) surfaces missing keys so translators can find them. Also generating labels at runtime and writing to localStorage on every miss is a subtle perf/side-effect smell."

Follow-up: "Why cache derived labels in localStorage?" (Avoid recomputing the string-transform each render — a micro-optimization.)

---

## Q12: The settings fetch blocks the whole app shell. Explain and fix.

Round: frontend. What it's testing: render-blocking data, loading UX.

Repo anchor: `frontend/src/apps/ErpApp.jsx:36-42` (fetch settings, render gated on `settingIsloaded`).

- **Junior:** "It loads settings first."
- **Mid:** "`ErpApp` dispatches `settingsAction.list` on mount and only renders the layout `if (settingIsloaded)` (`:42`). Until settings resolve, the user sees nothing but the loader — the shell is blocked on one request."
- **Senior:** "Settings gate currency/date formatting, so there's a real reason, but blocking the *entire* shell is heavy. I'd render the shell immediately with skeleton/placeholder formatting and hydrate settings when they arrive, or server-inline the critical settings so there's no round-trip. Also a slow settings endpoint here = whole-app outage; that's a single point of failure worth a timeout + fallback."

Follow-up: "What's the difference between this and Suspense-based data loading?" (This is imperative gating; Suspense would let you declaratively show a fallback per boundary.)

---

## Coverage map

Repo-anchored: Q1–Q12 (12/12). Concepts: render/commit, effect deps & stale closures, debounce + async races, unidirectional flow, token storage, code splitting + error boundaries, `Form.List`/keys, auth gating (client vs server), selector memoization, state placement, i18n, render-blocking data.
