# Language & Runtime Model

JavaScript and Node from first principles, anchored to IDURAR.

## The event loop and single-threadedness

**Concept:** Node runs your JavaScript on one thread. It doesn't do work in parallel; it *overlaps I/O waits*. When you `await` a database call, the thread is freed to handle other requests while the OS/driver does the I/O, and your function resumes (as a microtask) when the result arrives.

**Why it matters in production:** any *synchronous* CPU work blocks every request. One slow function freezes the whole server.

**Where the repo uses it well:** the list endpoint overlaps two independent I/O calls with `Promise.all` — the `find` and the `countDocuments` run concurrently, halving latency (`backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:32-50`). `isValidAuthToken.js:32-35` does the same for the user and password lookups.

**Where it has a sharp edge:** PDF generation renders synchronously on the request thread via `html-pdf` (`backend/src/controllers/pdfController/index.js:70-79`). While it renders, the event loop is blocked — every other user waits. And search builds an unbounded regex from user input (`createCRUDController/search.js:17`), a ReDoS vector that can pin the loop.

**Pitfall checklist:**
- Never do heavy CPU work (crypto loops, huge JSON, regex on untrusted input, image/PDF processing) on the request thread — offload to a worker or queue.
- `await` in a `for` loop serializes independent calls; use `Promise.all` when they're independent.
- `Promise.all` rejects on the first failure and doesn't cancel the rest.

**Drill:** find every `Promise.all` in the backend (`paginatedList.js:50`, `isValidAuthToken.js:35`) and, for each, confirm the two operations are truly independent. Then find the one place a loop should use `Promise.all` but doesn't.

**Interview angle:** "Explain the event loop" and "why does one slow request affect others?" → [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q3, Q7.

## Promises, async/await, microtasks

**Concept:** `async` functions return promises; `await` pauses the function and schedules its continuation on the **microtask queue**, which drains before the next macrotask (timer/IO callback). A rejected promise that's never `.catch()`ed is an unhandled rejection.

**Where the repo relies on it:** every controller is `async` and returns a promise. Because Express 4 doesn't await handlers, `catchErrors` (`backend/src/handlers/errorHandlers.js:9-32`) wraps each one to catch rejections — otherwise a thrown error in an async handler would leave the request hanging.

**Sharp edge:** fire-and-forget promises. `invoiceController/create.js:62-64` calls `increaseBySettingKey(...)` without `await`, so the HTTP response can return before it runs, and its errors are swallowed (`increaseBySettingKey.js:29-31`). That's a deliberate-looking but risky pattern.

**Pitfall checklist:**
- An un-awaited promise runs "sometime"; its errors won't reach your handler.
- `try/catch` only catches `await`ed rejections, not fire-and-forget ones.
- Mixing callbacks and promises (the PDF `toFile` callback, `pdfController/index.js:76-79`) needs care to not double-resolve.

**Drill:** in `invoiceController/create.js`, what happens to the client if `increaseBySettingKey` throws? (Nothing — it's un-awaited and self-swallows. The invoice still returns 200.) Now: what if you `await` it and it throws? (500 via `catchErrors`.)

**Interview angle:** Q1 (async error propagation), Q7 (`Promise.all` vs sequential).

## Closures

**Concept:** a function "closes over" the variables in scope where it was defined, keeping them alive after that scope returns.

**Where the repo depends on it:** the CRUD factory returns handlers that close over `Model` (`createCRUDController/index.js:20-31`) — that's how one factory produces per-entity controllers. The auth factory closes over `userModel` the same way (`createAuthMiddleware/index.js:7-35`).

**Sharp edge (the same feature, a bug):** React effects close over props/state at the time they run. `DataTable.jsx:162-168` has an effect with `[]` deps that closes over `entity` at mount — accidentally fine here because the component remounts per entity, but a stale-closure trap in general.

**Drill:** explain why the CRUD factory *needs* a closure (so each handler carries its own `Model` without re-looking-it-up per request) and why the React effect is a *risk* (captures a possibly-stale value).

**Interview angle:** Q5 (closures), Q2 frontend (effect deps).

## `this` and arrow functions

**Concept:** regular `function`s get `this` from how they're *called*; arrow functions capture `this` lexically from where they're *defined*.

**Where it matters:** Mongoose instance methods must be `function`s because Mongoose calls them with `this` bound to the document — `AdminPassword.js:38-45` (`generateHash`, `validPassword`) rely on `this.password`. An arrow function there would capture module scope and break.

**Where it's avoided:** the frontend is all function components and hooks — no `this` anywhere, which is part of why hooks are preferred over class components.

**Interview angle:** Q10 (`this`/arrows).

## Type coercion

**Concept:** JS auto-converts types across operators. `*`, `-`, `/` coerce to number; `+` prefers string concatenation if either side is a string.

**Where the repo leans on it (fragile):** `req.query.page` is a string used directly in `page * limit - limit` (`paginatedList.js:2-4`) — works because `*` coerces to number. If it were `+`, you'd get string concatenation. `items` is explicitly `parseInt`'d (`:3`) but `page` isn't.

**Pitfall checklist:**
- Validate/parse query params at the boundary; don't trust coercion.
- `'2abc' * 10` → `NaN`, and `NaN` propagates silently.
- Use `===` (the thunks do: `data.success === true`).

**Drill:** trace what `skip` becomes if a client sends `?page=abc`. (`'abc' * 10 - 10` → `NaN` → `.skip(NaN)`.) What would validate it cleanly? (Joi/zod at the boundary, or `Number.parseInt` with a guard.)

**Interview angle:** Q13 (coercion).

## Node module system (CJS)

**Concept:** the backend uses CommonJS — synchronous `require`, `module.exports`, `__dirname`. The frontend uses ESM — hoisted `import`, `export`, no `__dirname`.

**Where it shows:** `backend` uses `require('module-alias/register')` (`server.js:1`) to make `@/...` work at runtime. `frontend` is `"type":"module"` and Vite resolves `@` at build. They look identical but are wholly different mechanisms — which is why FE and BE can't share a file.

**Interview angle:** Q11 (module resolution).
