# Tooling & Build System

## The two toolchains

IDURAR has two independent toolchains because it's two independent packages (see [01-codebase-cartography/04-runtime-and-tooling-map.md](../01-codebase-cartography/04-runtime-and-tooling-map.md) for the raw map). This file is about *why* they're shaped that way and what it teaches.

### Backend: unbundled Node + module-alias

The backend ships source as-is — no bundler, no transpile. Node runs `src/server.js` directly. The one piece of tooling magic is `module-alias` (`server.js:1`): it patches Node's `require` at runtime so `@/helpers` resolves to `src/helpers` (configured in `backend/package.json` `_moduleAliases`).

**What this teaches:** path aliases aren't free. `module-alias` is a runtime monkey-patch, so any *other* tool (a test runner, a type checker, an IDE) needs its own alias config to understand `@/...`. Node's newer native `imports` field (subpath imports in `package.json`) is the modern, tool-agnostic alternative.

### Frontend: Vite

Vite serves ESM directly in dev (fast, no bundle) and uses Rollup to bundle for production (`vite build`). Config in `frontend/vite.config.js`:
- `@` alias resolved by Vite at build (`resolve.alias`) — a *different* mechanism than the backend's runtime alias, though the ergonomics match.
- Dev server on 3000 with a `/api` proxy to 8888.
- `@vitejs/plugin-react` for JSX + Fast Refresh.

**What this teaches:** dev and prod use different module strategies (unbundled ESM vs Rollup bundle), which is why a bug can appear only in the production build. `import.meta.env.PROD` branches (`serverApiConfig.js:1-16`) exist precisely because behavior differs between the two.

## The CJS/ESM split and why it matters

| | Backend | Frontend |
| --- | --- | --- |
| Module system | CommonJS (`require`/`module.exports`) | ESM (`import`/`export`, `"type":"module"`) |
| Alias resolved | at runtime (`module-alias`) | at build (Vite) |
| `__dirname` | available | not available (use `import.meta.url`) |
| Top-level await | no | yes |

**Consequence:** you cannot `require` a frontend ESM file from the backend or vice versa. Shared logic is duplicated — `backend/src/helpers.js` `calculate` and `frontend/src/utils/calculate.js` are the *same* money math written twice. To truly share you'd extract a third package with a build that emits both CJS and ESM, or move to ESM on the backend.

**Interview angle:** Q11 ([08/01](../08-interview-prep/01-js-ts-node-deep-dive.md)) — "how does `@/helpers` resolve?" You can now explain both mechanisms and why they can't share code.

## Linting: the only automated gate

`frontend/package.json` `lint`: `eslint . --ext js,jsx --report-unused-disable-directives --max-warnings 0`. Plugins: `react`, `react-hooks`, `react-refresh`. The `--max-warnings 0` means *any* warning fails — including `react-hooks/exhaustive-deps`, which would flag the effect-dependency issues in `DataTable.jsx:162-168` and `AutoCompleteAsync`.

There is **no backend lint, no Prettier script, no type-check, no test, no CI**. Prettier 3 is a devDependency but has no npm script — it's run manually or via editor integration.

**What this teaches:** know what your safety nets are before you change code. Here the net is one frontend lint command; everything else is manual verification. A change that "passes lint" has been checked for almost nothing behaviorally.

## npm specifics

- Two lockfiles (`backend/package-lock.json`, `frontend/package-lock.json`) — install in each folder.
- `engines` pins Node 20.9.0 / npm 10.2.4 — the `server.js:7-11` guard enforces Node ≥20 at boot.
- No workspace/monorepo tool, so no hoisting; each package has its own `node_modules`.

## Practical command reference

See [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md) for the full list with verified/inferred labels. The essentials:

```
# backend/
npm install
npm run dev          # nodemon, :8888
npm run setup        # seed (crashes in this copy — missing models)

# frontend/
npm install
npm run dev          # vite, :3000, proxies /api
npm run lint         # the one gate
npm run build        # production bundle
```

## Drill

Explain why editing only `backend/src/helpers.js` to fix a money-rounding bug would *not* fix the same bug in the browser. (The frontend uses its own copy, `frontend/src/utils/calculate.js`; they're duplicated because CJS and ESM can't share.) Then propose the smallest realistic way to deduplicate them and name the blocker.
