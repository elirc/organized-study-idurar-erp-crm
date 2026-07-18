# Behavioral STAR Stories

10 story worksheets sourced from studying this repo and from the tickets/projects in [06-contribution-practice](../06-contribution-practice/). For a real interview you'll swap in your *own* experiences — but if IDURAR is a project you genuinely worked through, these are legitimately yours to tell. Each is built to run **under two minutes** and **end on impact**.

STAR = Situation, Task, Action, Result. The two most common failures: no measurable Result, and burying the Action in setup. Rehearse until Action is 60% of the airtime and Result is concrete.

---

## Story 1: "Tell me about a complex codebase you had to learn."

Prompts this answers: learning ability, dealing with ambiguity, self-direction.
Source: studying this repo from zero.

- **Situation:** Inherited an ERP/CRM with a generated API — no explicit route file, ~370 source files, no tests, no docs.
- **Task:** Understand it well enough to change it safely and explain it.
- **Action:** Traced entry points top-down (`server.js` → `app.js` → generated routes), discovered the filename-driven convention (`models/utils/index.js`) that produces routes, mapped 7 end-to-end flows, and wrote anchored notes instead of trusting memory.
- **Result:** Could explain any request path from React to Mongo and back, and identified two confirmed defects (broken routes, non-atomic payments) within the first pass.
- **Evidence to cite:** the key-flows trace tables; the verification log.
- **Senior-signal detail:** "I refused to write a claim I couldn't anchor to a file and line — that discipline surfaced the bugs."
- **Resume bullet:** *Reverse-engineered an undocumented 370-file ERP codebase into a traced architecture map, uncovering two latent data-integrity bugs.*
- **Rehearsal check:** under 2 min? ends on the two bugs found? yes.

---

## Story 2: "Tell me about a subtle bug you found."

Prompts: attention to detail, debugging, technical depth.
Source: the non-atomic payment flow (Flow 4).

- **Situation:** Payment recording updated two documents — the payment and the parent invoice's cached balance.
- **Task:** Assess data-integrity risk before extending it.
- **Action:** Traced the three writes in `paymentController/create.js`, confirmed no transaction wraps them, and reasoned about the crash window: a failure after the payment insert but before the invoice `$inc` leaves `credit` permanently wrong.
- **Result:** Documented the exact broken invariant (`credit === SUM(payments)`), the retry double-count, and two fixes (transaction with replica-set prerequisite, or derive-on-read).
- **Evidence:** `paymentController/create.js:41,66-77`.
- **Senior-signal detail:** "I distinguished the crash-consistency bug from concurrency — the `$inc` is actually safe under concurrent increments; the real hole is the multi-write gap."
- **Resume bullet:** *Identified a cross-document consistency bug in payment processing and specified a transactional fix with a reconciliation guard.*
- **Rehearsal check:** ends on the fix + guard.

---

## Story 3: "Tell me about a time you disagreed with an approach."

Prompts: conflict, technical judgment, communication.
Source: a hard-delete review kata.

- **Situation:** A proposed change switched delete from soft (`removed:true`) to hard delete "to save space."
- **Task:** Push back without shutting the person down.
- **Action:** Showed that the whole app filters `removed:false` and that invoices reference payments — hard delete would orphan references and lose audit history. Offered an alternative: a purge job for old soft-deleted rows if storage was the real concern.
- **Result:** Kept the invariant intact; reframed the actual goal (storage) into a safer solution.
- **Evidence:** `createCRUDController/remove.js`, `invoiceController/remove.js:26-29`.
- **Senior-signal detail:** "I separated the stated solution from the underlying need and solved the need."
- **Resume bullet:** *Prevented a data-loss regression by reframing a 'hard delete' request into a background purge job.*

---

## Story 4: "Tell me about a security issue you caught."

Prompts: security awareness, ownership.
Source: unauthenticated download route (Flow 6).

- **Situation:** Invoice PDFs were served from a `/download` route with no authentication.
- **Task:** Judge whether it was acceptable and how to fix it.
- **Action:** Confirmed the route bypasses `isValidAuthToken` (`app.js:42`) because `window.open` can't send an auth header, and that it also ignored soft-delete. Proposed signed, expiring URLs issued from an authenticated endpoint as the minimal-blast-radius fix.
- **Result:** A concrete remediation that secures the file without breaking the open-in-new-tab UX.
- **Evidence:** `app.js:42`, `downloadPdf.js:9-11`, `DataTable.jsx:93-95`.
- **Senior-signal detail:** "I ranked fixes by blast radius and chose the standard one (signed URLs) over a disruptive rewrite."
- **Resume bullet:** *Closed an unauthenticated document-access gap using signed expiring URLs with zero UX disruption.*

---

## Story 5: "Tell me about a time you improved performance."

Prompts: performance, measurement.
Source: N+1 review kata / the aggregation flow.

- **Situation:** A dashboard stat could be computed by looping over clients and querying invoices per client.
- **Task:** Deliver the stat without the N+1.
- **Action:** Used a single aggregation with `$lookup` + `$facet` so all stats compute in one round-trip instead of N+1 queries (`clientController/summary.js:24-73`), and reasoned about the query count (1 vs 501 for 500 clients).
- **Result:** Constant-round-trip dashboard regardless of client count.
- **Evidence:** `clientController/summary.js:50-72`.
- **Senior-signal detail:** "I quantified the win in round-trips, not vibes, and noted it needs an index on the join field to stay fast."
- **Resume bullet:** *Cut a dashboard query from O(N) round-trips to one via MongoDB `$lookup`/`$facet` aggregation.*

---

## Story 6: "Tell me about a mistake you made."

Prompts: ownership, growth, honesty.
Source: an accidentally-correct effect (React deps).

- **Situation:** I initially defended a `useEffect` with an empty dep array that read a prop, assuming "runs once = safe."
- **Task:** Verify my own assumption.
- **Action:** Realized it was only *accidentally* correct — it worked because the component remounts per entity so the prop never changes mid-life; if it could change, the stale closure would bite. Corrected my mental model and flagged the honest fix (declare the dep or document the mount-only intent).
- **Result:** A more precise understanding of stale closures and a note for the next reader.
- **Evidence:** `DataTable.jsx:162-168`.
- **Senior-signal detail:** "The bug wasn't in the code — it was in my confidence. I now distinguish 'correct' from 'correct for a reason I verified.'"
- **Resume bullet:** *(behavioral only — no bullet)*

---

## Story 7: "Tell me about making a decision with incomplete information."

Prompts: ambiguity, pragmatism.
Source: choosing a fix order for the invoice-number race.

- **Situation:** Duplicate invoice numbers were possible via a client-side read race, with several possible fixes.
- **Task:** Decide what to ship first without a full concurrency test harness.
- **Action:** Ranked by blast radius: a unique compound index `{number, year}` enforces correctness at the DB immediately and is a one-line, reversible change; server-side atomic allocation is the better UX but a bigger change. Chose the index first, allocation second.
- **Result:** Correctness guaranteed fast; UX improvement scheduled.
- **Evidence:** `InvoiceForm.jsx:40`, `invoiceController/create.js:62-64`.
- **Senior-signal detail:** "I optimized for 'stop the bleeding safely now' over 'perfect later.'"
- **Resume bullet:** *Eliminated duplicate invoice numbers with a unique index as an immediate guard, then a server-side allocator.*

---

## Story 8: "Tell me about working with a messy/legacy codebase."

Prompts: pragmatism, respect for constraints.
Source: the whole repo.

- **Situation:** No tests, no types, magic-string state keys, duplicated FE/BE money logic.
- **Task:** Contribute without a risky big-bang refactor.
- **Action:** Scoped changes to small, testable-by-hand tickets that follow existing patterns (e.g., delete dead routes, add a Joi rule) instead of rewriting the reducer or introducing TypeScript wholesale. Documented risks in a register rather than fixing everything at once.
- **Result:** Safe incremental progress; a prioritized backlog for larger work.
- **Evidence:** the risk register; the good-first-tickets list.
- **Senior-signal detail:** "I resisted the urge to 'fix everything' — small blast radius beats heroic rewrites in code with no test net."
- **Resume bullet:** *Drove incremental improvements in a test-less legacy codebase via small, pattern-matching changes and a prioritized risk register.*

---

## Story 9: "Tell me about a time you had to explain something technical."

Prompts: communication, mentorship.
Source: writing this curriculum / explaining the generated-API convention.

- **Situation:** The API has no route file — endpoints are derived from model filenames, which confuses everyone new.
- **Task:** Make the convention understandable.
- **Action:** Traced the derivation (`Client.js` → `client` entity → `/api/client/*` → `clientController`) with a step table and a concrete "add a Widget model, get a Widget API" drill, so the reader learns by doing.
- **Result:** A newcomer can predict the API surface from the filesystem.
- **Evidence:** `models/utils/index.js:22-47`; the Flow 2 trace table.
- **Senior-signal detail:** "I taught the *mechanism* and its failure mode (renames break the API), not just the happy path."
- **Resume bullet:** *Documented an implicit filename-to-route convention into a hands-on onboarding drill.*

---

## Story 10: "Tell me about balancing quality and shipping."

Prompts: prioritization, judgment.
Source: triaging the risk register.

- **Situation:** Multiple issues surfaced at once — broken routes (crash), non-atomic payments (data), unauthenticated downloads (security), ReDoS (availability).
- **Task:** Order them for a realistic release.
- **Action:** Triaged by severity × likelihood: the crash blocks *everything* so it ships first (tiny fix); non-atomic payments and download auth are high-impact but need design; ReDoS is labeled *investigate* pending a proof. Sequenced accordingly.
- **Result:** A defensible release order that fixes the show-stopper immediately and schedules the deeper work.
- **Evidence:** the risk register.
- **Senior-signal detail:** "I didn't treat all bugs as equal — a one-line crash fix outranks a design-heavy consistency fix in sequencing even though the latter is 'scarier.'"
- **Resume bullet:** *Triaged a mixed bug backlog by severity × likelihood into a staged remediation plan.*

---

## Rehearsal protocol

For each story: say it aloud, time it. If over 2:00, cut the Situation. If the Result has no concrete noun (a number, a bug caught, a regression prevented), rewrite it. Map each to at least one common prompt category — conflict (3), mistake (6), ambiguity (7), technical depth (2), leadership/teaching (9) — so you can redirect any behavioral question to a story you've rehearsed.
