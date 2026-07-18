# Interview Prep

Target: **mid-level fullstack JS interviews** (React/Node/TS-adjacent). This module turns IDURAR into your portfolio of talking points. At least 60% of the question cards are anchored to real code in this repo, because the single biggest edge in a mid-level loop is answering with a *concrete example* instead of a memorized definition.

## How mid-level fullstack loops are usually structured

1. **Recruiter/technical screen** — a few concept questions, one small coding exercise. They're checking you're not lying on your resume.
2. **Technical deep-dive** — someone reads your background and asks "how does X work?" and "why did you choose Y?" This is where repo anchors win.
3. **Practical coding** — build or extend a small feature, often CRUD. IDURAR *is* this exercise; you've built its shape.
4. **System design** (increasingly common at mid-level) — "design a simplified [product]." See [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md).
5. **Behavioral** — STAR stories. See [06-behavioral-star-stories.md](06-behavioral-star-stories.md).

## The golden rule

Every answer has three layers. Interviewers grade which layer you reach:

- **Definition** (junior): "A JWT is a signed token." — necessary, not sufficient.
- **Tradeoff + concrete example** (mid): "IDURAR signs a JWT *and* stores it in `loggedSessions` so it can revoke on logout — the cost is a DB read per request." — this is the hire signal.
- **Failure mode + alternative** (senior-leaning): "…but that array grows unbounded; I'd use a `tokenVersion` integer or short-lived access + refresh tokens instead." — this is what gets you the higher band.

Reach for layer 2 on every question and layer 3 whenever you can. Never stop at layer 1.

## The files

| File | Round | Cards |
| --- | --- | --- |
| [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) | JS/TS/Node | 15 |
| [02-frontend-framework-questions.md](02-frontend-framework-questions.md) | Frontend/React | 12 |
| [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) | API/Data | 13 |
| [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) | System design | 1 full walkthrough + 5 variations |
| [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) | Debugging + review | 4 debug + 4 review sims |
| [06-behavioral-star-stories.md](06-behavioral-star-stories.md) | Behavioral | 10 STAR worksheets |
| [07-two-week-cram-plan.md](07-two-week-cram-plan.md) | Plan | day-by-day |

That's 40 discrete question cards plus a full system-design walkthrough, timed simulations, and 10 stories — over 50 practice prompts, the majority anchored to code you've read.

## How to practice

- **Say it aloud, timed.** 90 seconds per card. If you ramble past two minutes you don't know it yet.
- **Trace the anchor first.** Before answering a repo-anchored card, open the file. Your answer should reference what you actually see.
- **Record layer 3.** For each card, write your one-sentence "failure mode + alternative." That sentence is what separates you from the other candidates.
- **Rehearse the STAR stories until they're under two minutes and end on impact.**

Start here if your interview is soon: [07-two-week-cram-plan.md](07-two-week-cram-plan.md).
