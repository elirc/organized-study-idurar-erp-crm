# Two-Week Interview Cram Plan

A day-by-day plan for a candidate with a mid-level fullstack interview in two weeks, using this repo as the study lab. ~2 hours/day. Every day pairs *input* (read/study) with *output* (say it aloud or write it) — passive reading doesn't stick under interview pressure.

Self-assessment checkpoints at Day 7 and Day 14.

---

## Week 1 — Build the foundation and the anchors

### Day 1 — Orientation
- Do [00-fast-track.md](../00-fast-track.md) steps 1–3 (run it or read the flows if no DB).
- Output: teach-back the request lifecycle aloud (fast-track step 5), 5 min, no notes.

### Day 2 — The two spine flows
- Read [01-codebase-cartography/05-key-flows.md](../01-codebase-cartography/05-key-flows.md) Flows 1 (login) and 4 (payment).
- Output: whiteboard the payment consistency problem from memory; state the invariant and the fix.

### Day 3 — JS/TS/Node cards, first half
- Study [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) Q1–Q8. Open every anchor.
- Output: record yourself answering Q2 (money), Q4 (JWT), Q8 (server-authoritative) in 90 s each. Listen back — did you reach layer 2?

### Day 4 — JS/TS/Node cards, second half
- Study Q9–Q15.
- Output: answer Q9 (NoSQL/ReDoS), Q14 (autopopulate N+1), Q15 (invoice-number race) aloud. Write the layer-3 sentence for each.

### Day 5 — Frontend cards
- Study [02-frontend-framework-questions.md](02-frontend-framework-questions.md) all 12. Open the AutoCompleteAsync and DataTable anchors.
- Output: answer Q3 (debounced search race), Q5 (localStorage token), Q10 (state placement) aloud.

### Day 6 — API/data cards
- Study [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) all 13.
- Output: answer Q4 (IDOR scoping), Q5 (transactions), Q12 (download auth), Q13 (mass assignment) aloud.

### Day 7 — Checkpoint + first mock
- **Timed debugging round:** do [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) Debug 1 and Debug 3 under a 10-min timer each, narrating the reproduce→narrow→fix method.
- **Self-assessment (grade yourself against [09-reference/learning-rubrics.md](../09-reference/learning-rubrics.md)):**
  - Can I trace login and payment end-to-end with anchors? (Y/N)
  - Can I reach layer 2 on 80% of the JS/frontend/API cards? (Y/N)
  - Can I narrate a debugging process without jumping to a guess? (Y/N)
- If two or more are "N", spend an extra day here before Week 2.

---

## Week 2 — System design, review, stories, integration

### Day 8 — System design walkthrough
- Study [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) Steps 1–5.
- Output: whiteboard the full invoicing design in 25 min, out loud, standing up. Hit requirements → API → model → consistency → scaling.

### Day 9 — System design variations
- Study the 5 variation prompts (multi-tenant, real-time, 10x, recurring, audit).
- Output: whiteboard "now make it multi-tenant" in 10 min. This is the highest-signal variation — it tests whether you caught the single-tenant assumption.

### Day 10 — Code review round
- Do all 4 review katas in [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) under 8-min timers.
- Output: for each, write one blocking finding as a *kind, specific* PR comment. Review-round interviewers grade tone as much as correctness.

### Day 11 — Debugging round 2
- Do Debug 2 and Debug 4 timed.
- Output: narrate Debug 4 (ReDoS) to a rubber duck; emphasize why one slow regex stalls all users (event loop).

### Day 12 — Behavioral stories
- Study [06-behavioral-star-stories.md](06-behavioral-star-stories.md). Pick your 5 strongest (suggest 1, 2, 3, 4, 7).
- Output: rehearse each under 2:00, ending on impact. Record and trim anything over time.

### Day 13 — Full mock loop
- Simulate a loop in one sitting: 3 rapid JS/API cards (15 min) → 1 system design (30 min) → 1 debugging (10 min) → 2 behavioral (5 min). No notes.
- Output: note every question where you stalled at layer 1; drill those anchors tonight.

### Day 14 — Final checkpoint + light review
- Re-answer the cards you flagged on Day 13.
- Light review only — do not cram new material the day before.
- **Final self-assessment:**
  - System design: can I do Steps 1–4 in 20 min with anchors? (target: Y)
  - Can I name a failure mode + alternative (layer 3) on the 10 most important cards? (target: Y)
  - Are all 5 STAR stories under 2 min and ending on impact? (target: Y)
  - Can I narrate a debugging and a review round calmly? (target: Y)

---

## The 10 must-know anchors (memorize these file:line references)

If you remember nothing else, remember where these live — they answer the majority of the loop:

1. `backend/src/app.js:39-43` — router topology + auth placement.
2. `backend/src/models/utils/index.js:22-47` — filename→route convention.
3. `backend/src/controllers/appControllers/paymentController/create.js:41-77` — non-atomic two-doc write (the money story).
4. `backend/src/controllers/middlewaresControllers/createAuthMiddleware/isValidAuthToken.js:45-53` — dual-token session.
5. `backend/src/controllers/appControllers/invoiceController/create.js:30-48` — server-authoritative math + ownership.
6. `backend/src/helpers.js:47-60` — decimal-safe money.
7. `backend/src/controllers/appControllers/clientController/summary.js:50-72` — `$lookup` (N+1 avoidance).
8. `backend/src/controllers/middlewaresControllers/createCRUDController/paginatedList.js:32-50` — offset pagination + parallel queries.
9. `frontend/src/redux/erp/reducer.js:70-127` — generic `keyState` reducer.
10. `frontend/src/request/request.js:16-25` — how the token reaches every request.

## Golden-rule reminder

For every answer: definition (junior) → **tradeoff + concrete anchor (mid)** → failure mode + alternative (senior). Aim for the middle layer every time and reach for the third whenever you can. That single habit is what moves you from "answered the question" to "hire."
