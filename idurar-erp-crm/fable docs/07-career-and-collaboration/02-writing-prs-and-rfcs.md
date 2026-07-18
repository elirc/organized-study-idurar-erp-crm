# Writing PRs and RFCs

## PR descriptions that get merged

A good PR description answers four questions before the reviewer has to ask. Structure:

```
## What
One or two sentences: the change, in plain terms.

## Why
The problem this solves. Link the issue/ticket. If it fixes a bug, describe the bug.

## How
The approach and any non-obvious decisions. Call out what you considered and rejected.

## Testing
How you verified it. In this repo (no test suite), that means the manual steps you ran,
or the new test you added. Be specific: "logged in, created an invoice for a new client,
recorded a $40 payment on a $100 invoice, confirmed status became 'partially'."

## Risks / follow-ups
Blast radius, what could break, what you're leaving for later.
```

**Example, tailored to this repo (Ticket 9, summary date fix):**

```
## What
Restore date-range scoping to the invoice summary endpoint.

## Why
`type=month` returned the all-time total because the date `$match` was commented out
(invoiceController/summary.js:37-40) while startDate/endDate were computed but unused.
The dashboard's "this month" figure was wrong.

## How
Re-enabled the date `$match` using the existing computed range. Confirmed the
totalInvoice facet is meant to be scoped by reading the frontend dashboard's usage.

## Testing
Seeded invoices in June and July. GET /api/invoice/summary?type=month now returns only
July's total; ?type=year returns both. Verified manually against the DB.

## Risks / follow-ups
Low blast radius — one aggregation stage. If any consumer expected the all-time total
from this endpoint, they'd see a change; grep found none. Follow-up: add a test once the
harness lands (#9 in mid-level tickets).
```

Why this works: the reviewer can verify the claim (anchors), sees the reasoning, knows how it was tested, and knows the risk. Compare to "fixed summary bug" — which forces the reviewer to reverse-engineer everything.

## Commit messages

- Subject line: imperative, ≤50 chars — "Restore date scoping in invoice summary."
- Body (if needed): the *why*, wrapped at ~72 chars. The diff shows the *what*.
- One logical change per commit. Don't mix a bug fix with a refactor.

## When to write an RFC instead of just a PR

Write an RFC (design doc) before coding when the change:
- alters a **contract** (the API envelope, the filename convention, the response shape),
- has a **wide blast radius** (touches the generic CRUD factory, auth, or every entity),
- is **hard to reverse** (a schema migration, a new dependency, an infra change),
- or has **multiple viable approaches** worth discussing (transactions vs derive-on-read).

Small, local, reversible changes just need a good PR.

## RFC template (tailored to this repo)

```
# RFC: <title>

## Problem
What's broken or missing, with anchors. Why it matters now.

## Goals / Non-goals
What this does and explicitly does NOT do.

## Proposed design
The approach. For this repo, address:
- Does it change the API envelope or the filename→route contract?
- Does it touch the generic CRUD factory (affects all entities)?
- Does it need a replica set / new infra (transactions)?
- FE/BE duplication implications (CJS/ESM boundary)?

## Alternatives considered
At least one, with why you rejected it (e.g. derive-on-read vs transactions).

## Migration plan
Deploy order, backfill, backward compatibility. Note: no migration framework exists yet.

## Test plan
Given no suite exists: what harness you'd add, what cases you'd cover.

## Rollout / rollback
Feature flag? Canary? How to undo it.

## Risks & open questions
Blast radius, unknowns, what would make a reviewer reject it.
```

**Worked mini-RFC (securing downloads):**

> **Problem:** `/download` serves invoice PDFs with no auth (`app.js:42`) and ignores soft-delete (`downloadPdf.js:9-11`). Any leaked URL is valid forever.
> **Proposed design:** an authenticated endpoint issues a signed, expiring URL (HMAC of id+exp); the download route verifies the signature before serving; add `removed:false`.
> **Alternatives:** a token query param (simpler, but no expiry); full session cookie for downloads (breaks `window.open` more). Signed URL is the smallest blast radius.
> **Migration:** add the signing endpoint; update the frontend's `window.open` to fetch a URL first; keep the old route briefly, then remove.
> **Test:** forged/expired signatures rejected; deleted invoice 404s.
> **Rollback:** feature-flag; revert to the open route.
> **Rejection criteria:** if it breaks existing bookmarked download links without a deprecation window.

## Interview angle

"How do you communicate a technical change?" — show the PR structure and when you'd escalate to an RFC. The judgment ("small reversible = PR, contract-changing = RFC") is the senior signal. See [08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md).

## Drill

Take any mid-level ticket ([06/02](../06-contribution-practice/02-mid-level-feature-tickets.md)) and decide: PR or RFC? Then write whichever applies for it, using the template. Self-grade — Strong: you justified the PR-vs-RFC call by blast radius and reversibility, and your Testing section is concrete given the no-suite reality.
