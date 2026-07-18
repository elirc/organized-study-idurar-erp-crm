# 05 — Quality Engineering

How to make this codebase — which has **no tests, no CI, and no types** — safer to change, and the transferable engineering judgment that goes with it.

1. [01-testing-strategy.md](01-testing-strategy.md) — what test layers this repo should have and what to test at each.
2. [02-writing-tests-here.md](02-writing-tests-here.md) — concrete recipes to write the *first* tests, with exact setup.
3. [03-systematic-debugging.md](03-systematic-debugging.md) — the reproduce→narrow→fix method on 5 real scenarios.
4. [04-performance-thinking.md](04-performance-thinking.md) — where the hotspots are and how to find them.
5. [05-security-checklist.md](05-security-checklist.md) — the security map + a pre-merge checklist.
6. [06-observability-and-operations.md](06-observability-and-operations.md) — "how would I know this broke?" per flow.

The recurring theme: this repo has almost no safety net, so quality here is about *building* the net (first tests, first metrics) and *reasoning carefully* in its absence, not about polishing an existing one.
