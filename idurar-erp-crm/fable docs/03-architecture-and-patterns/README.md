# 03 — Architecture & Patterns

How IDURAR is structured and why, and what it commits you to.

1. [01-boundaries-and-layers.md](01-boundaries-and-layers.md) — the layers, what each owns, and where boundaries leak.
2. [02-data-model-and-persistence.md](02-data-model-and-persistence.md) — schema, relationships, indexes, transactions, safe schema change.
3. [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) — validation layers, authn vs authz, the IDOR gap.
4. [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) — side effects, idempotency, the non-atomic payment problem.
5. [05-pattern-catalog.md](05-pattern-catalog.md) — **14 pattern cards** for recognizing these shapes anywhere.
6. [06-architecture-critique.md](06-architecture-critique.md) — strengths, risks, and "what I'd change owning this for 3 months." Doubles as system-design interview prep.

The transferable skill: reading architecture as a set of *commitments*. Every boundary, every denormalized field, every convention buys something and costs something. Seniors and interviewers both want to hear the cost, not just the name.
