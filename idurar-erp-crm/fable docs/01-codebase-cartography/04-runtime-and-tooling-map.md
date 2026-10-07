# Runtime & Tooling Map

## Runtimes and their boundaries

| Surface | Runtime | Module system | Entry | Port |
| --- | --- | --- | --- | --- |
| Backend | Node 20 (`backend/package.json:8-11`) | CommonJS (`require`) | `src/server.js` | 8888 |
| Frontend dev | Vite 5 on Node | ESM (`"type":"module"`) | `src/main.jsx` | 3000 |
| Frontend prod | browser | ESM bundle | built `dist/` | served statically |

The two packages **cannot share code directly** despite both using an `@` alias — the backend's `@` is resolved at runtime by `module-alias` (`server.js:1`), the frontend's by Vite at build time (`vite.config.js`). Different mechanisms, incompatible module systems. Any shared logic (like money math) is *duplicated*, not imported (`backend/src/helpers.js` vs `frontend/src/utils/calculate.js`).

## Package manager and scripts

npm (lockfiles are `package-lock.json`; `engines` pins npm 10.2.4).

**Backend** (`backend/package.json`):
| Script | Command | Purpose |
| --- | --- | --- |
| `start` | `node src/server.js` | production run |
| `dev` | `nodemon src/server.js` | watch mode |
| `setup` | `node src/setup/setup.js` | seed admin + settings (**crashes in this copy** — missing models) |
| `upgrade` | `node src/setup/upgrade.js` | migrations-ish |
| `reset` | `node src/setup/reset.js` | wipe |

No `test`, no `lint` script on the backend.

**Frontend** (`frontend/package.json`):
| Script | Command | Purpose |
| --- | --- | --- |
| `dev` | `vite` | dev server :3000 |
| `build` | `vite build` | production bundle |
| `lint` | `eslint . --ext js,jsx --max-warnings 0` | the only automated gate in the repo |
| `preview` | `vite preview` | serve the built bundle |
| `dev:remote` | `cross-env VITE_DEV_REMOTE=remote vite` | point dev UI at a remote backend |

## The dev proxy

In dev, the browser calls `http://localhost:8888/api/` directly (`frontend/src/config/serverApiConfig.js:1-4`), and Vite also proxies `/api` → `:8888` (`vite.config.js` `server.proxy`). CORS is wide open on the backend (`app.js:21-26`, `origin:true, credentials:true`).

## Environment variables (high level — no secrets)

Backend reads from `.env` and `.env.local` (`server.js:14-15`). Referenced in code:

| Var | Used at | Purpose |
| --- | --- | --- |
| `DATABASE` | `server.js:17` | Mongo connection string (**required** — missing = the friendly error at `:21-26`) |
| `JWT_SECRET` | `authUser.js:19`, `isValidAuthToken.js:22` | signs/verifies tokens (**required** for auth) |
| `PORT` | `server.js:36` | defaults to 8888 |
| `OPENAI_API_KEY` | `server.js:19` | read but lightly used |
| `PUBLIC_SERVER_FILE` | `pdfController/index.js:59` | base for PDF asset URLs |
| `NODE_ENV` | `package.json` production script | error verbosity |
| `RESEND_API` | (email deps) | transactional email (Resend) |

Frontend (Vite, `VITE_`-prefixed, `serverApiConfig.js`):

| Var | Purpose |
| --- | --- |
| `VITE_BACKEND_SERVER` | remote backend base URL |
| `VITE_DEV_REMOTE` | `'remote'` to use the remote backend in dev |
| `VITE_FILE_BASE_URL` | file asset base |

A minimum working `.env` for the backend: `DATABASE=mongodb://localhost:27017/idurar` and `JWT_SECRET=<any-string>` (both **inferred** from the code paths above, not from a checked-in template — `backend/` has no committed `.env.example`).

## Build/lint/type tooling

- **Bundler:** Vite + `@vitejs/plugin-react` (frontend only; backend is unbundled Node).
- **Lint:** ESLint 8 with `eslint-plugin-react`, `react-hooks`, `react-refresh` (frontend). `--max-warnings 0` means any warning fails the lint. No backend lint.
- **Format:** Prettier 3 (frontend devDep; no script, run manually).
- **Types:** none. `@types/react` is present but there's no `tsconfig`, no `.ts` files, no type-check step. This is plain JS throughout.
- **Tests:** none. No test runner in either `package.json`, no `*.test.*` / `*.spec.*` files.
- **CI:** no build/test CI. `.github/workflows/` does exist, but it holds only `codesee-arch-diagram.yml` (a CodeSee architecture-diagram action on push/PR, `continue-on-error: true`), `github-repo-stats.yml` (a daily stats cron) and a stray `CODEOWNERS` file. None of them installs dependencies, lints, builds or tests. (A `CODEOWNERS` inside `workflows/` is not read by GitHub; it must live in `.github/`, the repo root or `docs/`, which makes it a small good-first fix.)

The practical consequence: **the only automated safety net is the frontend ESLint gate.** Every backend change and every cross-stack behavior is verified by hand. Treat that as context for every contribution — "add a test" often means "add the *first* test," which is a bigger task (see [05-quality-engineering/02-writing-tests-here.md](../05-quality-engineering/02-writing-tests-here.md)).

## Drill

List, from memory, the two environment variables without which the backend cannot function, and what fails if each is missing. (Answers: `DATABASE` → Mongo connection error at boot; `JWT_SECRET` → login signs with `undefined` and token verification throws.) Then state the one automated gate that exists and what it checks.
