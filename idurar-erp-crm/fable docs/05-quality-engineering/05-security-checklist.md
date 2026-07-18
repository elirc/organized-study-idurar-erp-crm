# Security Checklist

The security surfaces of IDURAR mapped to real code, then a pre-merge checklist. Findings are **confirmed** unless labeled *investigate*.

## Threat map

| Threat | Present? | Location | Status |
| --- | --- | --- | --- |
| Broken access control / IDOR | latent | no per-record scoping (`paginatedList.js:32-41`) | fine single-tenant; critical if multi-user |
| Unauthenticated resource access | **yes** | `/download` route (`app.js:42`, `downloadPdf.js`) | confirmed — serves business PDFs, ignores soft-delete |
| NoSQL injection | mitigated | list `equal` object guard (`paginatedList.js:19-29`) | confirmed mitigation |
| ReDoS | *investigate* | `new RegExp(req.query.q)` (`search.js:17`) | shape present, payload unproven |
| Mass assignment | **yes** | generic update spreads `req.body` (`createCRUDController/update.js:9`) | confirmed |
| XSS → token theft | exposure | JWT in localStorage (`redux/auth/actions.js:20`) | confirmed exposure |
| CSRF | low | token in header (not cookie) so CSRF is limited | by design |
| Path traversal | mitigated | `/public` containment check (`corePublicRouter.js:24-29`) | confirmed mitigation |
| Info disclosure | **yes** | raw `error` echoed in responses (`errorHandlers.js:22-28`) | confirmed |
| Secrets in code | no | env-based (`JWT_SECRET`, `DATABASE`) | ok |
| Dependency risk | **yes** | `html-pdf`/PhantomJS deprecated | confirmed |
| Rate limiting | **no** | `express-rate-limit` is a dependency but not applied in `src` (grep finds no usage) | confirmed gap — login is unthrottled |
| Unbounded growth | **yes** | `loggedSessions` never pruned (`authUser.js:23`) | confirmed |

## The top findings explained

### Unauthenticated PDF download (confirmed, high)
`/download/:dir/:file` is mounted with no auth middleware (`app.js:42`). Anyone with (or who guesses/leaks) the URL gets the PDF, and `downloadPdf.js:9-11` doesn't filter `removed:false`, so deleted invoices are still downloadable. ObjectIds are hard to guess (~96 bits) but obscurity ≠ authorization — leaked/logged URLs stay valid forever. **Fix:** signed, expiring URLs issued from an authenticated endpoint; add the `removed` filter. See [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q12.

### Mass assignment in generic update (confirmed, medium)
`createCRUDController/update.js:9` does `findOneAndUpdate(id, req.body)` — the entire client body becomes the update. A client could set `removed`, `createdBy`, `credit`, or any field. The invoice/payment overrides are safer (they reconstruct the update explicitly). **Fix:** whitelist updatable fields per entity.

### Token in localStorage (confirmed exposure, medium)
`redux/auth/actions.js:20` stores the JWT in localStorage; `request.js:20-24` reads it. Any XSS exfiltrates it. **Mitigations:** short token TTL, strict CSP, or httpOnly cookie + CSRF (bigger change — Kata 3 in [04/04](../04-code-reading-gym/04-review-katas.md)).

### Info disclosure (confirmed, low-medium)
Every 500 (and the production error handler) echoes the raw `error` object into the JSON body (`errorHandlers.js:22-28,71-77`), potentially leaking stack traces, query internals, or connection details. **Fix:** in production, return a generic message and log the detail server-side only.

### ReDoS (investigate, medium)
`search.js:17` and `paginatedList.js:15` build regexes from raw `q`. A catastrophic-backtracking pattern could pin the event loop. The *shape* is confirmed; a working payload wasn't demonstrated. **Fix:** escape to a literal, cap length, or use `$text`.

## Pre-merge security checklist

Run this on every PR that touches routes, controllers, or queries:

- [ ] **New route:** is it under an authenticated mount, or intentionally public (and documented as such)?
- [ ] **Reads a record by id:** would it need an ownership/tenant filter in a multi-user world?
- [ ] **Writes from `req.body`:** are fields whitelisted, or is the whole body spread (mass assignment)?
- [ ] **Trusts client for money/identity:** is it recomputed/stamped server-side?
- [ ] **Query from user input:** is any regex escaped and length-capped? Is an object rejected where a primitive is expected?
- [ ] **Soft-deletable model:** does the query include `removed:false`?
- [ ] **Error responses:** do they avoid leaking the raw error/stack in production?
- [ ] **New dependency:** is it maintained? (Don't add another `html-pdf`.)
- [ ] **Secrets:** nothing hardcoded; env-based only.
- [ ] **File serving:** path canonicalized + containment-checked?

## Interview angle

Security questions at mid-level usually probe: authn vs authz (you have the IDOR story), injection (NoSQL guard + ReDoS gap), and secure defaults (the positional-auth fragility). Lead with a concrete anchor every time — "the download route at `app.js:42` has no auth middleware" beats "you should always authenticate." See [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).

## Drill

Pick the mass-assignment finding. Write the whitelist fix for `createCRUDController/update.js` and explain why the invoice controller's explicit `update.js` doesn't have the same problem. Then decide: should the fix go in the generic controller (affects all entities) or per-entity? State the blast-radius tradeoff.
