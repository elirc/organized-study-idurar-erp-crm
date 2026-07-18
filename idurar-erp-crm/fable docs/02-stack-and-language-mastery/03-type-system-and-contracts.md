# Type System & Contracts

This repo is **plain JavaScript** — no TypeScript, no `tsconfig`, no `.ts` files (though `@types/react` is installed as a leftover). That absence is itself a teaching subject: contracts here are enforced at *runtime* (Joi, Mongoose) or *not at all* (magic strings, filename conventions), never at compile time.

## What "contract" means

A **contract** is a promise about shape that two pieces of code rely on. Contracts can be enforced at three moments:

| When | Mechanism in this repo | Example |
| --- | --- | --- |
| Compile time | none (no TS) | — |
| Runtime, at the boundary | Joi | `invoiceController/schemaValidate.js:2-24` |
| Runtime, at persistence | Mongoose schema | `Invoice.js` `required`/`enum` |
| Never (implicit) | conventions | filename→route (`models/utils/index.js:22-47`), `keyState` strings |

## The contracts that exist

**Joi request validation** (`invoiceController/schemaValidate.js`): the strongest, most explicit contract in the codebase. It declares that `client` is a string-or-object, `items` is a non-empty array where each item has `itemName`/`quantity`/`price`/`total`, etc. A violation returns 400 before any DB work. But it's applied *only* to invoices (and payments/auth) — generic CRUD entities have no Joi, so their contract is whatever Mongoose enforces.

**Mongoose schema** (all models): shape + `required` + `enum` + `default`. Enforced on `save` and on `findOneAndUpdate` when `runValidators:true` (`createCRUDController/update.js:12`). This is the last line of defense.

**The response envelope** (`{success, result, message}`): an *unwritten* contract. Every endpoint honors it and the client depends on it (`redux/erp/actions.js:47`), but nothing enforces it — a new endpoint could return a bare object and only break at runtime in the client.

## The contracts that are missing — and what breaks

Three bugs in *this copy* trace directly to the absence of compile-time contracts:

1. **`routes.jsx` renders undefined components.** `<Quote/>`, `<PaymentMode/>`, `<Taxes/>` are referenced (`routes.jsx:73-116`) with no imports. TypeScript (or even ESLint `no-undef`) would flag "Cannot find name 'Quote'" at build. Instead it's a runtime `ReferenceError` that crashes the app. This is the canonical "types would have caught it."

2. **The filename→route contract is stringly-typed across the whole stack.** `Invoice.js` → `/api/invoice` → `entity:'invoice'` in the frontend. Rename the model and nothing tells you the frontend's hardcoded strings are now wrong until requests 404. No type connects them.

3. **`keyState` magic strings.** `dispatch({..., keyState:'list'})` — a typo writes to a new state key silently (`redux/erp/reducer.js:83-87`). A union type `'list'|'create'|'read'|...` would reject `'lst'` at compile time.

## What TS / JSDoc / a shared schema would add

- **TypeScript:** compile-time checks on the three bugs above, plus IDE autocomplete on the invoice document shape and the Redux state tree. Cost: a build step and migrating ~370 files.
- **JSDoc + `checkJs`:** a lighter path — add `@typedef` comments and `// @ts-check`, get much of the safety without renaming files. Realistic for incremental adoption here.
- **A shared schema (Zod/Joi + type inference):** define the invoice shape once, derive both the runtime validator and the static type, and share it FE/BE — closing the envelope-contract gap. The blocker is the CJS/ESM split (see [04-tooling-and-build-system.md](04-tooling-and-build-system.md)); you'd need a build to share.

## `unknown` vs `any` (interview staple)

Even though this repo has neither, you must be able to contrast them:
- `any` disables the type checker — it's an escape hatch that silently propagates.
- `unknown` is the *safe* top type — you can hold anything but must narrow (check) before using it.
- For an API response of unverified shape, type it `unknown` and validate (with Joi/Zod) before use — which is conceptually what `schemaValidate.js` does at runtime.

## Pitfall checklist

- No types means every refactor is find-and-replace with no safety net — lean on tests (which also don't exist here) or careful manual verification.
- Runtime validation only protects the paths that have it; audit which entities have Joi and which rely on Mongoose alone.
- Implicit contracts (filenames, magic strings) are the most dangerous because nothing signals when they break.

## Drill

Pick the `keyState` typo scenario. Write the TypeScript union type that would prevent it, and the JSDoc `@typedef` equivalent that would prevent it without converting the file to `.ts`. Then find one more implicit contract in the repo that types would make explicit.

**Interview angle:** Q6 ([08/01](../08-interview-prep/01-js-ts-node-deep-dive.md)) — "name three bugs TypeScript would have caught here." You have three concrete, anchored answers.
