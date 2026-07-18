# 01 — Codebase Cartography

Maps for navigating IDURAR. Read in this order:

1. [01-system-map.md](01-system-map.md) — the shape of the repo: monorepo layout, ownership, public vs private surfaces.
2. [02-file-reading-order.md](02-file-reading-order.md) — 25 files in the order that makes the system click, with junior/mid/senior paths.
3. [03-domain-glossary.md](03-domain-glossary.md) — the business nouns (invoice, credit, paymentStatus…) with code locations.
4. [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) — package manager, scripts, dev servers, env vars, runtime boundaries.
5. [05-key-flows.md](05-key-flows.md) — **the centerpiece**: 7 flows traced end to end with trace tables, risks, and interview angles. Start here if you only read one file.

The cartography answers "where is everything and how is it shaped." The transferable skill is *how to map any unfamiliar repo fast*: find the entry point, follow the routing, find where business logic lives, find where data is persisted, and find the trust boundaries. Every file here models that method on IDURAR.
