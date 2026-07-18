# Systematic Debugging

The method, then 5 repo-specific scenarios. The method is what an interviewer grades in a debugging round — narrate it out loud, every step.

## The method

1. **Reproduce** — get a reliable trigger. A bug you can't reproduce, you can't verify fixed.
2. **Narrow** — is it in layer A or B? Bisect the pipeline (frontend vs backend, controller vs model).
3. **Hypothesize** — one specific, falsifiable guess.
4. **Test cheaply** — the smallest probe that confirms/refutes (a log, a breakpoint, a query, a single request).
5. **Fix the root cause** — not the symptom.
6. **Add a regression guard** — a test (or, absent a test suite, a documented check) so it can't silently return.

The tools you actually have here: browser DevTools (Network + React DevTools), `console.log` (used liberally already, e.g. `CreateItem.jsx:90`), Node `--inspect` / debugger, and MongoDB query logging / Compass. No CI logs (no CI), no APM.

---

## Scenario 1: "An invoice shows credit that doesn't match its payments."

Symptom: `invoice.credit` ≠ sum of its non-removed payments.

- **Reproduce:** record a payment; simulate a failure after the payment insert (throw in `create.js` before `:66`, or kill the process).
- **Narrow:** is the payment row present? (Yes.) Is `credit` updated? (No.) → the gap is between the two writes.
- **Hypothesize:** non-atomic multi-write; a failure after write #1 leaves the invariant broken.
- **Test cheaply:** grep for `startSession`/`withTransaction` — none. Query the DB: payment exists, invoice `credit` stale.
- **Fix root cause:** wrap writes in a transaction (`paymentController/create.js:41,66-77`), or derive `credit` on read.
- **Regression guard:** the reconciliation test from [02-writing-tests-here.md](02-writing-tests-here.md) Recipe 1, plus a failure-injection variant.
- **Senior lesson:** the bug is *timing*, not logic — the arithmetic is right, the atomicity is wrong.
- **Interview version:** "I'd reproduce with a failure injected between the two writes, confirm no transaction wraps them, and fix with `withTransaction` (noting it needs a replica set) or derive-on-read."

---

## Scenario 2: "The app is a white screen after `npm run dev`."

Symptom: blank page, console `ReferenceError: Quote is not defined`.

- **Reproduce:** load `/`; read the console.
- **Narrow:** stack points at `routes.jsx`. Frontend, module-eval time — not a data problem.
- **Hypothesize:** a referenced component isn't imported.
- **Test cheaply:** search `routes.jsx` for `const Quote` (no match) vs `<Quote` (match at `:73`).
- **Fix root cause:** delete the dead route entries for `Quote`/`PaymentMode`/`Taxes` (`routes.jsx:73-116`) and matching nav links.
- **Regression guard:** none automatable without tooling; argue for ESLint `no-undef` / TypeScript, which would catch it at build.
- **Senior lesson:** a runtime `ReferenceError` for a top-level identifier is almost always a missing import/type gap — the fastest class of bug to localize.
- **Interview version:** "Read the stack, confirm the identifier is referenced but never declared, remove the dead code, and note that a type-check step would have caught it at build."

---

## Scenario 3: "Two invoices got the same number."

Symptom: duplicate `number` within a year.

- **Reproduce:** open the create form in two tabs; both show `last+1`; submit both.
- **Narrow:** where does `number` originate? Client form seeds it from a setting at load (`InvoiceForm.jsx:40`), not from the server at submit.
- **Hypothesize:** two clients read the same stale `last_invoice_number` → collision; the server bump is un-awaited (`create.js:62`).
- **Test cheaply:** confirm no unique `{number,year}` index (schema has none); confirm the bump swallows errors (`increaseBySettingKey.js:29-31`).
- **Fix root cause:** atomic server-side allocation, and/or unique index with retry.
- **Regression guard:** concurrent-create integration test asserting distinct numbers.
- **Senior lesson:** "read-modify-write on the client" is always a race; move allocation to the DB.
- **Interview version:** covered in [08-interview-prep/05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md) Debug 3.

---

## Scenario 4: "Dashboard 'this month' total looks too high."

Symptom: the monthly invoice total equals the all-time total.

- **Reproduce:** call `/api/invoice/summary?type=month`; compare to `type=year`; they match.
- **Narrow:** backend aggregation. Read `invoiceController/summary.js`.
- **Hypothesize:** the date filter isn't applied.
- **Test cheaply:** read the `$match` at `:33-42` — the `date` filter is **commented out** (`:37-40`), while `startDate`/`endDate` are computed (`:27-29`) but unused.
- **Fix root cause:** uncomment/repair the date `$match`, using the computed range.
- **Regression guard:** test seeding invoices across two months, asserting `type=month` excludes the other.
- **Senior lesson:** validated input (`type` is whitelisted) that's then *ignored* is a sneaky correctness bug — validation presence doesn't mean the value is used.
- **Interview version:** "The param is validated but the `$match` that should apply it is commented out, so scoping silently does nothing — I'd restore it and add a cross-month test."

---

## Scenario 5: "Search sometimes makes the whole API unresponsive."

Symptom: certain search strings spike CPU; all requests stall.

- **Reproduce:** send a pathological pattern to `/api/client/search?q=...`; watch CPU/latency.
- **Narrow:** one request stalls *all* others → event-loop blocking, not a slow DB.
- **Hypothesize:** ReDoS via `new RegExp(req.query.q)` (`search.js:17`).
- **Test cheaply:** time a known evil pattern locally (label *investigate* until shown).
- **Fix root cause:** escape input to a literal, cap length, or use a `$text` index.
- **Regression guard:** test asserting a crafted input returns within a time budget.
- **Senior lesson:** on a single-threaded runtime, one CPU-bound request is a availability incident — untrusted regex and heavy sync work are the usual culprits.
- **Interview version:** [08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md) Debug 4.

---

## Anti-patterns to avoid (and to *not* do in an interview)

- Changing code before reproducing — you won't know if you fixed it.
- Fixing the symptom (add a try/catch to hide the crash) instead of the cause.
- "Shotgun debugging" — changing five things at once.
- Skipping the regression guard — the bug returns and you look like you never fixed it.

## Drill

Take Scenario 4. Before reading the fix, predict from `summary.js:27-42` what the `type=week` total would be, and why. Then write the one-line change that restores date scoping and the test that proves it.
