# Writing Tests Here

Concrete recipes for the *first* tests in this repo. All setup is **inferred** (no test infra exists to run against); the code shapes are real. Adapt package versions to what installs cleanly.

## One-time setup (backend)

```bash
# backend/
npm i -D jest supertest mongodb-memory-server
```

`backend/jest.config.js` (inferred):
```js
// Illustrative fake code: not from this repo
module.exports = {
  testEnvironment: 'node',
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' }, // mirror module-alias for Jest
  setupFilesAfterEnv: ['<rootDir>/test/setup.js'],
};
```

`backend/test/setup.js` (inferred) — the module-alias trap: Jest doesn't know `@/` unless you map it (above), because `module-alias` is a runtime patch Jest bypasses.
```js
// Illustrative fake code: not from this repo
const { MongoMemoryReplSet } = require('mongodb-memory-server');
const mongoose = require('mongoose');
let mongo;
beforeAll(async () => {
  mongo = await MongoMemoryReplSet.create({ replSet: { count: 1 } }); // replSet so transactions work later
  process.env.JWT_SECRET = 'test-secret';
  await mongoose.connect(mongo.getUri());
  require('glob').globSync('./src/models/**/*.js').forEach((f) => require(require('path').resolve(f)));
});
afterEach(async () => {
  for (const c of Object.values(mongoose.connection.collections)) await c.deleteMany({});
});
afterAll(async () => { await mongoose.disconnect(); await mongo.stop(); });
```

A tiny data builder (`test/factory.js`, inferred):
```js
// Illustrative fake code: not from this repo
const mongoose = require('mongoose');
exports.makeInvoice = (over = {}) => mongoose.model('Invoice').create({
  number: 1, year: 2026, client: new mongoose.Types.ObjectId(),
  createdBy: new mongoose.Types.ObjectId(), date: new Date(), expiredDate: new Date(),
  currency: 'usd', items: [{ itemName: 'A', quantity: 1, price: 100, total: 100 }],
  total: 100, subTotal: 100, credit: 0, discount: 0, ...over,
});
```

---

## Recipe 1 — Happy path: record a payment updates the invoice

Tests the core invariant. Note: calling the controller directly needs a fake `req`/`res` because logic is coupled to Express (see [testing-strategy](01-testing-strategy.md)).

```js
// Illustrative fake code: not from this repo
const create = require('@/controllers/appControllers/paymentController/create');
const { makeInvoice } = require('../factory');
const mongoose = require('mongoose');

const mockRes = () => { const r = {}; r.status = jest.fn(() => r); r.json = jest.fn(() => r); return r; };

test('recording a payment increases invoice.credit and updates paymentStatus', async () => {
  const invoice = await makeInvoice({ total: 100, discount: 0, credit: 0 });
  const req = { body: { invoice: invoice._id, client: invoice.client, amount: 40, number: 1, date: new Date() }, admin: { _id: new mongoose.Types.ObjectId() } };
  const res = mockRes();

  await create(req, res);

  const updated = await mongoose.model('Invoice').findById(invoice._id);
  expect(updated.credit).toBe(40);
  expect(updated.paymentStatus).toBe('partially');
  expect(res.status).toHaveBeenCalledWith(200);
});
```

## Recipe 2 — Validation failure: overpayment is rejected

```js
// Illustrative fake code: not from this repo
test('paying more than the balance is rejected with 202', async () => {
  const invoice = await makeInvoice({ total: 100, discount: 0, credit: 0 });
  const req = { body: { invoice: invoice._id, client: invoice.client, amount: 150 }, admin: { _id: new mongoose.Types.ObjectId() } };
  const res = mockRes();
  await create(req, res);
  expect(res.status).toHaveBeenCalledWith(202);
  const count = await mongoose.model('Payment').countDocuments();
  expect(count).toBe(0); // no payment written on rejection
});
```

## Recipe 3 — Server-authoritative totals (invoice create)

```js
// Illustrative fake code: not from this repo
const invoiceCreate = require('@/controllers/appControllers/invoiceController/create');
test('server recomputes total, ignoring client-supplied total', async () => {
  const req = { body: { client: new mongoose.Types.ObjectId(), number: 5, year: 2026, status: 'draft',
    date: new Date(), expiredDate: new Date(), taxRate: 0,
    items: [{ itemName: 'A', quantity: 2, price: 50, total: 999 }], total: 0.01 },
    admin: { _id: new mongoose.Types.ObjectId() } };
  const res = mockRes();
  await invoiceCreate(req, res);
  const result = res.json.mock.calls[0][0].result;
  expect(result.subTotal).toBe(100);  // 2*50, not the client's 999/0.01
});
```

## Recipe 4 — Auth boundary (integration via supertest)

Requires exporting the app; `backend/src/app.js` already `module.exports = app`.
```js
// Illustrative fake code: not from this repo
const request = require('supertest');
const app = require('@/app');
test('protected route without a token returns 401', async () => {
  const res = await request(app).get('/api/invoice/list');
  expect(res.status).toBe(401);
});
```

## Recipe 5 — Soft-delete cascade

```js
// Illustrative fake code: not from this repo
const invoiceRemove = require('@/controllers/appControllers/invoiceController/remove');
test('deleting an invoice soft-deletes its payments', async () => {
  const invoice = await makeInvoice();
  await mongoose.model('Payment').create({ invoice: invoice._id, client: invoice.client,
    createdBy: invoice.createdBy, number: 1, amount: 10, currency: 'usd' });
  const res = mockRes();
  await invoiceRemove({ params: { id: invoice._id } }, res);
  const p = await mongoose.model('Payment').findOne({ invoice: invoice._id });
  expect(p.removed).toBe(true);
});
```

## Recipe 6 — Frontend component test (money formatting)

```bash
# frontend/
npm i -D vitest @testing-library/react jsdom
```
```jsx
// Illustrative fake code: not from this repo
import { render, screen } from '@testing-library/react';
// mock the settings selector so useMoney has a format config, then assert a rendered amount
// (the point: test the formatting behavior, not Ant Design internals)
```

## Wiring it into a gate

Add to `backend/package.json`: `"test": "jest"`. Then a minimal CI (GitHub Actions) runs `npm test` in `backend/` and `npm run lint && npm run build` in `frontend/` on every PR — the safety net this repo lacks. That CI file is a good-first-ticket / senior project.

## Drill

Run Recipe 1 mentally against the real `paymentController/create.js`: does `paymentStatus` become `'partially'` for a 40-on-100 payment? Trace `create.js:59-64` to confirm the arithmetic. Then predict what Recipe 1 would show *after* you introduce the transaction fix, and why the replica-set `MongoMemoryReplSet` in setup matters.
