# Architectural Cartographer

This suite is a top-down map of the IDURAR ERP/CRM codebase. It is written for a developer who wants to understand the system well enough to explain it, debug it, review changes, and eventually own features.

Use it in this order:

1. `00-reading-map.md` gives the mental model, the files to read first, and the review red flags.
2. `01-junior-engineer.md` teaches setup, folders, entry points, component anatomy, backend anatomy, and the domain glossary.
3. `02-mid-level-engineer.md` traces architecture, state, API contracts, and a full invoice/payment flow.
4. `03-senior-engineer.md` critiques the codebase and turns the critique into refactoring, testing, security, and performance work.
5. `04-reference-suite.md` is the permanent desk reference for onboarding, architecture, review, debugging, change planning, and interviews.

How to work checkpoints:

Read the linked files first, answer the questions without looking at the rubric, then use the self-grade section immediately below each checkpoint. A strong answer should name specific files, line ranges, and the reason the system is shaped that way.

Important reading rule:

Every factual claim in this suite points to a concrete source file and line range. When a pattern does not exist, the docs say so directly and explain the effect of the gap.

