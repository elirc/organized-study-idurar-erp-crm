# System Map

## Repo shape

IDURAR is a **two-package monorepo without a workspace tool** — `backend/` and `frontend/` are independent npm projects with their own `package.json`, `node_modules`, and lockfile. There is no root `package.json`, no npm/yarn/pnpm workspace, no Turborepo. You install and run each separately. This is the simplest possible "monorepo": two folders in one git repo.

```
idurar-erp-crm/
├── backend/          Express 4 + Mongoose 8 JSON API (CommonJS, Node 20)
│   └── src/
│       ├── server.js         boot: connect Mongo, glob models, listen :8888
│       ├── app.js            Express app: middleware + 5 router mounts
│       ├── routes/           coreRoutes (auth, admin, settings, download, public)
│       │                     + appRoutes (generated per-entity CRUD)
│       ├── controllers/      coreControllers, appControllers, middlewaresControllers
│       ├── models/           coreModels (Admin, Setting…) + appModels (Client, Invoice, Payment)
│       ├── middlewares/      auth, uploads, settings
│       ├── handlers/         errorHandlers, downloadHandler
│       ├── pdf/ + emailTemplate/  Pug templates
│       └── setup/            seed scripts
└── frontend/         React 18 + Ant Design 5 + Redux Toolkit (ESM, Vite)
    └── src/
        ├── main.jsx → RootApp.jsx → apps/IdurarOs.jsx   bootstrap + auth gate
        ├── router/           AppRouter (authed) + AuthRouter (login) + routes.jsx
        ├── redux/            auth, erp (generic entity store), settings
        ├── request/          axios wrapper (the one network chokepoint)
        ├── modules/          ErpPanelModule (generic CRUD UI) + Invoice/Payment modules
        ├── components/       reusable widgets (DataTable, AutoCompleteAsync…)
        ├── pages/            thin per-route configs
        └── settings/ locale/ money + i18n
```

## The request topology (ASCII)

```
Browser (React :3000)
   │  axios, Bearer token from localStorage        frontend/src/request/request.js
   ▼
Vite dev proxy  /api → :8888                         frontend/vite.config.js
   ▼
Express app                                          backend/src/app.js
   ├─ /api  (coreAuth)          login/logout/reset   ← NO auth middleware (public)
   ├─ /api  (coreApi)           admin, settings      ← isValidAuthToken
   ├─ /api  (erpApi)            invoice/payment/...  ← isValidAuthToken
   ├─ /download/:dir/:file      PDF generation       ← NO auth (public!)
   └─ /public/:sub/:dir/:file   static files         ← NO auth, path-traversal guard
   ▼
Controllers ── generic CRUD factory OR custom override
   ▼
Mongoose models ── MongoDB
```

The three unauthenticated mounts (`/api` auth routes, `/download`, `/public`) are the security-relevant boundaries. `/download` being public is a real finding (see [05-key-flows.md](05-key-flows.md) Flow 6).

## Ownership map

| Concern | Owns | Location | Must NOT own |
| --- | --- | --- | --- |
| HTTP routing | URL → controller mapping | `backend/src/routes/` | business logic |
| Generic CRUD | list/read/create/update/delete/search | `controllers/middlewaresControllers/createCRUDController/` | entity-specific math |
| Business logic | invoice totals, payment/credit invariant | `controllers/appControllers/{invoice,payment}Controller/` | HTTP framing (leaks a bit — see critique) |
| Data shape | schemas, validation, refs | `backend/src/models/` | request handling |
| Auth | login, JWT, session allow-list | `controllers/middlewaresControllers/createAuthMiddleware/` | — |
| Client routing | route table, auth gate | `frontend/src/router/`, `apps/IdurarOs.jsx` | data fetching |
| Server state | entity lists/current item | `frontend/src/redux/erp/` | UI-only state |
| Network | every API call, token injection | `frontend/src/request/request.js` | rendering |
| Presentation | tables, forms, formatting | `frontend/src/modules/`, `components/`, `pages/` | business rules (mostly display-only math) |

## Public interfaces vs private internals

**Public (the contracts other code depends on):**
- The REST API surface generated in `backend/src/routes/appRoutes/appApi.js:8-26` — 9 routes per entity.
- The response envelope `{success, result, message}` — every endpoint honors it.
- The `frontend/src/request/request.js` object — the only sanctioned way to call the API.
- The `config`-driven component contract (`DataTable({config})`).

**Private (implementation details, change freely):**
- The CRUD factory internals (`createCRUDController/*`).
- The generic reducer's `keyState` mechanism.
- PDF rendering, upload storage, settings loading.

## Where a senior looks first

1. `app.js` — what's authenticated, what isn't (the 5 mounts).
2. The data models — the domain in ~5 files.
3. The one flow that mutates money across documents (`paymentController`) — where the integrity risk lives.
4. Test/CI presence — here, absent, which reframes every change as "manually verified."

## Drill

Draw the request topology from memory, then mark each mount as authenticated or not. Check against `app.js:39-43`. Self-grade — Solid: you got the 3 public mounts right. Strong: you can explain *why* `/download` has to be public given `window.open`, and why that's still a problem.
