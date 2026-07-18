# IDURAR Upskill Curriculum

A training lab built on this repository for one learner: a junior fullstack JS engineer (React/Node) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles. Every page teaches two things at once: **where things live in this repo** and **the transferable pattern you can name in any repo — and in an interview**.

## What this repo is

IDURAR is an open-source ERP/CRM: a single-tenant invoicing app where an admin logs in, manages clients, issues invoices, records payments against them, and downloads PDF copies. The backend is Express 4 + Mongoose 8 on Node 20 (`backend/package.json:8-11`), a single process serving a JSON API under `/api` plus unauthenticated `/download` and `/public` file routes (`backend/src/app.js:39-43`). The frontend is React 18 + Ant Design 5 + Redux Toolkit, built with Vite (`frontend/package.json`), talking to the API through one shared axios wrapper (`frontend/src/request/request.js`). There is no test suite, no CI config, and no TypeScript — which makes it a realistic specimen: most interview stories come from codebases like this, not from pristine ones.

**Important:** this copy is trimmed relative to upstream IDURAR. The Quote, PaymentMode, and Taxes verticals were removed from the backend models, but the frontend still references them (`frontend/src/router/routes.jsx:73-87`, `frontend/src/apps/Navigation/NavigationContainer.jsx` quote/taxes menu items) and `backend/src/setup/setup.js:54-67` still requires the deleted models. Several drills in this curriculum use those seams deliberately. Treat them as confirmed defects of *this copy*, documented in [09-reference/risk-register.md](09-reference/risk-register.md).

## Learning tracks

| Track | Folder | What you get |
| --- | --- | --- |
| Cartography | [01-codebase-cartography](01-codebase-cartography/README.md) | System map, reading order, glossary, 7 traced flows |
| Stack mastery | [02-stack-and-language-mastery](02-stack-and-language-mastery/README.md) | Node/JS runtime, React/Redux mental models, contracts, tooling |
| Architecture | [03-architecture-and-patterns](03-architecture-and-patterns/README.md) | Layers, data model, auth, side effects, 14 pattern cards, critique |
| Reading gym | [04-code-reading-gym](04-code-reading-gym/README.md) | Annotation drills, trace tables, fake-code contrasts, review katas |
| Quality | [05-quality-engineering](05-quality-engineering/README.md) | Testing strategy, debugging method, performance, security, ops |
| Contribution | [06-contribution-practice](06-contribution-practice/README.md) | 16 junior tickets, 10 mid tickets, 6 senior projects, katas |
| Career | [07-career-and-collaboration](07-career-and-collaboration/README.md) | Review mindset, PR/RFC writing, maintainer communication |
| **Interview prep** | [08-interview-prep](08-interview-prep/README.md) | 50+ question cards anchored to this repo, system design, STAR stories, cram plan |
| Reference | [09-reference](09-reference/) | Commands, risk register, rubrics, verification log |

## Recommended paths

- **One weekend:** [00-fast-track.md](00-fast-track.md) end to end.
- **Two weeks:** fast track → `01/05-key-flows.md` → `03/05-pattern-catalog.md` → one junior ticket from `06/01` → `08/04-system-design-from-this-repo.md`.
- **Eight weeks:** modules in order 01 → 02 → 03 → 04 → 05, doing one ticket from 06 per week, finishing with all of 08.
- **Brand-new junior:** 00 → 01 (all) → 02 (all) → the junior halves of 04's drills; only then 03.
- **Junior who knows React/Node:** 00 → 01/05-key-flows → 03 (all) → 04 (all) → 06 mid tickets.
- **Mid-level, new to the repo:** 01/01 + 01/05, then straight to 03/06-architecture-critique and 06/03-senior-build-projects.
- **Senior doing architecture review:** 03/06-architecture-critique.md and 09-reference/risk-register.md, verify against code.
- **Interview in two weeks:** go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md). It schedules everything else you need.

## Conventions used everywhere

- **Anchors.** Every claim about this codebase cites `path:lines`, e.g. `backend/src/app.js:39-43`. Paths are relative to the repo root (`idurar-erp-crm/`). Line numbers were confirmed against the file at writing time and logged in [09-reference/verification-log.md](09-reference/verification-log.md). If a file has drifted, trust the pattern description and re-locate.
- **Fake code.** Any snippet not from this repo begins with `// Illustrative fake code: not from this repo`. Everything else is real.
- **Drills and self-grading.** Most sections end with a drill and a Basic/Solid/Strong rubric. Do the drill before reading the rubric; grade yourself honestly — the rubric describes *observable* behaviors, not vibes.
- **Uncertainty labels.** Claims we could not fully verify are marked *investigate* or *possible risk*, never asserted as bugs.
- **Interview angle.** Sections flag when a concept is common interview material and link into `08-interview-prep/`.
- **Senior vocabulary.** Words like *invariant*, *boundary*, *contract*, *idempotency*, *blast radius* are used where they genuinely apply and defined in context the first time. They are interview vocabulary — collect them.

## The mindset ladder

- A **junior** asks: *how do I make it work?*
- A **mid-level** asks: *is this the right pattern? What breaks if the input is hostile, the network is slow, or two requests race?*
- A **senior** asks: *what does this commit us to, who pays the cost, and how do we reduce the risk?*

Interviews for mid-level roles test exactly the second and third questions. When an interviewer asks "how would you design invoice payments?", they are checking whether you see the consistency invariant between `Payment.amount` and `Invoice.credit` (`backend/src/controllers/appControllers/paymentController/create.js:56-77`) — not whether you can write a POST handler. This curriculum trains you to answer with evidence from code you have actually read.
