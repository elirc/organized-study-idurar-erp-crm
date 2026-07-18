# Testing Strategy

## The starting point: zero tests

There is no test runner in either `package.json`, no `*.test.*`/`*.spec.*` files, and no CI (confirmed in the verification log). So "add a test" here often means "add the *first* test," which includes choosing and wiring a framework. That's a bigger task than in a repo with an existing suite — and a legitimate senior project.

## What test layers this repo should have

| Layer | What it tests | Tooling (recommended) | Priority here |
| --- | --- | --- | --- |
| Unit | pure logic (money math, status derivation) | Jest/Vitest | blocked by coupling — see below |
| Integration (API) | a route end-to-end against a real-ish DB | Jest + supertest + `mongodb-memory-server` | **highest value** |
| Component | React components in isolation | Vitest + Testing Library | medium |
| E2E | full user flows in a browser | Playwright | later |

## Why unit tests are hard here (and what to do)

The business logic is coupled to Express — controllers take `(req, res)` and both compute and respond (`invoiceController/create.js`). You can't unit-test the total-computation without constructing fake `req`/`res`. The `calculate` helpers (`helpers.js:47-60`) are the *one* thing that's purely unit-testable today.

**So the highest-ROI first tests are integration tests** at the API layer — they test the real behavior (validation, money math, the payment invariant) without needing to refactor. Once you extract a service layer ([03-architecture/01-boundaries-and-layers.md](../03-architecture-and-patterns/01-boundaries-and-layers.md)), unit tests become cheap.

## What to test — priority order

1. **The payment invariant** (`paymentController/create.js`): create an invoice, record a payment, assert `invoice.credit` and `paymentStatus` update correctly; record an overpayment, assert it's rejected. This is the riskiest logic, so it earns the first test.
2. **Server-authoritative totals** (`invoiceController/create.js`): send a bogus client `total`, assert the server recomputes.
3. **Auth boundary** (`isValidAuthToken`): no token → 401; valid token → 200; logged-out token → 401.
4. **Validation** (`schemaValidate.js`): missing required field → 400; empty items → 400.
5. **Soft-delete cascade** (`invoiceController/remove.js`): delete invoice, assert its payments are soft-deleted too.

## What NOT to test

- Ant Design internals, Mongoose internals, `currency.js` — you don't own them.
- Trivial getters, the generated route wiring itself (test *behavior* through it, not the glob).
- Exact log strings, formatting minutiae that change often.
- The 40+ README translations.

## Fixtures, builders, isolation

- **Fixtures/builders:** since seeding is fragile (`setup.js` crashes), tests should build their own data with small factory functions (`makeInvoice(overrides)`), not rely on `npm run setup`.
- **Isolation:** each test needs a clean DB. `mongodb-memory-server` spins an in-memory Mongo per run; clear collections in `afterEach`. Never test against a shared dev DB.
- **Time/randomness:** the invoice number uses `Date.getFullYear()` and settings; freeze time (Jest fake timers) and stub the counter so tests are deterministic.
- **Flake prevention:** avoid real network (the `mail` endpoint is a stub anyway); avoid `setTimeout`-based assertions; await all promises.

## The transactions caveat for tests

If you write the payment-invariant test *after* adding transactions, `mongodb-memory-server` must run as a replica set (it supports `replSet` mode) for `withTransaction` to work. Note this — it's a common "my test passes locally but the transaction code path isn't actually exercised" trap.

## Interview angle

"How would you add testing to a legacy codebase with none?" Strong answer: start with integration tests at the API layer (highest behavior coverage per unit of effort, no refactor needed), pick the riskiest logic first (the payment invariant), then extract a service layer to unlock cheap unit tests. Name the isolation tooling (`mongodb-memory-server`) and the replica-set caveat. See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md).

## Drill

Write the test *plan* (not code yet — that's the next file) for the payment invariant: list the arrange/act/assert for the happy path, the overpay rejection, and the `amount:0` rejection. State what you'd stub and what you'd use a real in-memory DB for.
