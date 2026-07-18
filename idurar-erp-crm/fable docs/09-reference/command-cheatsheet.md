# Command Cheatsheet

Every command marked **verified** (executed while writing this curriculum) or **inferred** (derived from manifests/docs, not run — no MongoDB was available). All are read from `backend/package.json`, `frontend/package.json`, `frontend/vite.config.js`, and `INSTALLATION-INSTRUCTIONS.md`.

## Backend (`backend/`)

| Task | Command | Status |
| --- | --- | --- |
| Install deps | `npm install` | inferred |
| Run (watch) | `npm run dev` | inferred (nodemon, port 8888) |
| Run (prod) | `npm start` | inferred |
| Seed DB | `npm run setup` | inferred — **crashes** in this copy (missing PaymentMode/Taxes models, `setup.js:54-67`) |
| Upgrade | `npm run upgrade` | inferred |
| Reset | `npm run reset` | inferred |
| Lint | — | none (no backend lint script) |
| Test | — | none (no test script) |

Required env (`backend/.env`, **inferred** from code):
```
DATABASE=mongodb://localhost:27017/idurar   # server.js:17
JWT_SECRET=<any-secret-string>              # authUser.js:19, isValidAuthToken.js:22
PORT=8888                                   # optional, server.js:36
```

## Frontend (`frontend/`)

| Task | Command | Status |
| --- | --- | --- |
| Install deps | `npm install` | inferred |
| Run dev | `npm run dev` | inferred (Vite, port 3000, proxies /api → 8888) |
| Run dev against remote backend | `npm run dev:remote` | inferred |
| Build | `npm run build` | inferred |
| Preview build | `npm run preview` | inferred |
| Lint (the one gate) | `npm run lint` | inferred (`eslint . --ext js,jsx --max-warnings 0`) |
| Format | `npx prettier --write .` | inferred (Prettier is a devDep, no script) |

Default login after a successful seed: `admin@admin.com` / `admin123` (`setup.js:18-26`).

## Exploration commands used while writing this curriculum (verified)

| Task | Command | Status |
| --- | --- | --- |
| Repo root | `git rev-parse --show-toplevel` | verified |
| Git state | `git status --short` (no commits yet on `main`) | verified |
| Backend file count | `find backend/src -type f \| wc -l` → 116 | verified |
| Frontend file count | `find frontend/src -type f \| wc -l` → 258 | verified |
| Directory maps | `find <dir> -type d \| sort` | verified |
| Confirm rate-limit unused | `rg 'rate-limit\|rateLimit'` in `backend/src` → no hits | verified |

## Useful investigation commands (inferred — for the learner)

| Task | Command |
| --- | --- |
| Find all API-relevant queries missing `removed` | `rg 'findOne\(\|find\(' backend/src \| rg -v removed` |
| Turn on Mongoose query logging (in code) | `mongoose.set('debug', true)` |
| Explain a query (mongo shell) | `db.invoices.find({removed:false}).explain('executionStats')` |
| Find raw money math | `rg '\.total\s*[-+]\s*\|\.discount\s*[-+]' backend/src` |
| List routes generated per entity | read `backend/src/routes/appRoutes/appApi.js:9-17` |

## Docker / codegen / migrations

- **Docker:** no Dockerfile or compose referenced in the source flows (a docker-compose single-node replica set would be needed for the transaction work — see [06/03 Project 1](../06-contribution-practice/03-senior-build-projects.md)).
- **Codegen:** none.
- **Migrations:** none (no `migrate-mongo` or equivalent; `setup`/`upgrade`/`reset` are seed helpers, not a migration framework).

## Why so many "inferred"

No MongoDB instance was available in the authoring environment, so nothing that requires a running DB (setup, dev servers, the app itself) was executed. The commands are read directly from the manifests and are correct as *commands*; their runtime behavior (especially the setup crash and the frontend route crash) was determined by reading source, and is documented as **confirmed by reading** in [risk-register.md](risk-register.md) and [verification-log.md](verification-log.md).
