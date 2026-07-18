# Learning Rubrics

Self-assessment rubrics with **observable behaviors**, not vague traits. For each skill, find the highest row where you can honestly do everything described. The "Interview-ready" column is what a mid-level interviewer wants to hear.

## How to use

Rate yourself per skill monthly. "I understand X" isn't observable; "I can trace X end-to-end citing files" is. Only claim a level if you can *demonstrate* the behavior, ideally out loud or in writing.

---

## Skill: Reading an unfamiliar codebase

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Can find where a feature's code lives by searching | "I'd grep for the route and follow it" |
| Mid | Traces a request from UI to DB citing specific files; identifies the entry point and conventions | Can walk through IDURAR's request lifecycle with anchors |
| Senior | Maps trust boundaries, invariants, and risks on first read; predicts where bugs hide | "First I'd check auth placement, the money flow, and whether there are tests" |

## Skill: JavaScript / Node fundamentals

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Uses async/await; knows `===` vs `==` | Defines the event loop |
| Mid | Explains microtasks, `Promise.all` vs sequential, closures, coercion with a concrete example | Reaches "tradeoff + example" on every JS card in [08/01](../08-interview-prep/01-js-ts-node-deep-dive.md) |
| Senior | Predicts event-loop blocking; explains why one slow regex stalls all requests; names when the standard answer is wrong | Reaches "failure mode + alternative" (layer 3) unprompted |

## Skill: React / frontend

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Builds components with hooks; wires state and props | Explains render vs commit |
| Mid | Spots stale-closure and dependency-array bugs; knows where state should live; explains selector memoization | Answers the frontend cards ([08/02](../08-interview-prep/02-frontend-framework-questions.md)) with repo anchors |
| Senior | Identifies async races, derived-state duplication, render-blocking data; proposes RTK Query-style fixes | Critiques a component's architecture, not just its bugs |

## Skill: API & data modeling

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Designs CRUD routes; knows embed vs reference exists | Lists REST verbs |
| Mid | Justifies embed-vs-reference; explains validation layers, pagination, indexing | Answers the API cards ([08/03](../08-interview-prep/03-api-and-data-modeling-questions.md)) with tradeoffs |
| Senior | Reasons about transactions, idempotency, IDOR, migration safety; names prerequisites (replica set) | Leads the system-design walkthrough ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)) fluently |

## Skill: Debugging

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Adds logs; fixes the symptom they see | Describes trying things |
| Mid | Follows reproduce→narrow→hypothesize→fix→guard; bisects layers | Narrates the method calmly in a timed round |
| Senior | Distinguishes root cause from symptom; adds a regression guard; explains the class of bug | Turns a debug into a systemic insight ([08/05](../08-interview-prep/05-debugging-and-code-review-rounds.md)) |

## Skill: Code review

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Finds one surface issue; comments on style | "This looks wrong" |
| Mid | Separates blocking/important/optional; cites the pattern a change breaks; kind + specific comments | Reviews a kata with correct severity and tone |
| Senior | Identifies the systemic fix (centralize, don't patch); reviews design, not just code | Catches the invariant/security issue others miss |

## Skill: Security awareness

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Knows auth exists; hashes passwords | "Always authenticate" |
| Mid | Distinguishes authn vs authz; spots mass assignment, IDOR, injection with anchors | Answers security cards citing real code |
| Senior | Thinks in trust boundaries and blast radius; proposes minimal-blast-radius fixes | "The download route at `app.js:42` has no auth — here's the signed-URL fix" |

## Skill: Testing & quality

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Writes a test when a suite exists | "Tests are good" |
| Mid | Chooses the right layer; isolates DB/time; tests the risky logic first | "I'd start with integration tests on the payment invariant" |
| Senior | Builds the first harness; extracts code to make it testable; names isolation tooling + caveats | Proposes tests + CI as the enabler for all other work |

## Skill: Communication & collaboration

| Level | Observable behavior | Interview-ready |
| --- | --- | --- |
| Junior | Asks for help; writes basic PR descriptions | — |
| Mid | Asks with investigation + one precise question; writes what/why/how/testing PRs; disagrees with evidence | Tells a clean STAR conflict story |
| Senior | Knows when to RFC vs PR; disagrees-and-commits; helps others without solving it for them | Frames a change by blast radius and reversibility |

---

## Overall readiness checklist (mid-level interview)

- [ ] I can trace 3+ IDURAR flows end-to-end with file anchors.
- [ ] I reach "tradeoff + concrete example" on ≥80% of the [08](../08-interview-prep/) cards.
- [ ] I reach "failure mode + alternative" on the 10 must-know anchors ([08/07](../08-interview-prep/07-two-week-cram-plan.md)).
- [ ] I can do the system-design walkthrough (Steps 1–4) in 20 minutes.
- [ ] I have 5 STAR stories under 2 minutes each, ending on impact.
- [ ] I can narrate a debugging round and a review round calmly.
- [ ] I can name this repo's top 5 risks and their fixes from memory.

When every box is checked, you're ready. Track progress against this list weekly during the cram plan.
