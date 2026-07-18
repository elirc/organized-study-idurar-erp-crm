# User Story Build Path

This suite gives you ten real feature tickets to build inside this codebase. Each story is grounded in the app's actual domain: customers, invoices, payments, settings, navigation, and backend persistence.

Difficulty progression:

Stories 1-3 are easy UI changes that touch one or two files. Stories 4-5 read existing data and reuse current modules. Stories 6-7 add API/state behavior. Stories 8-9 are full-stack features that touch frontend, backend, and database. Story 10 is an expert design story with architectural implications.

How to know a story is done:

- The acceptance criteria can be verified manually.
- The affected route still loads.
- The relevant backend endpoint still returns the expected JSON.
- The UI handles loading, success, and failure states in the same style as nearby code.
- You can explain the files touched and why.

How to use AI without getting the solution handed to you:

Use this prompt:

```text
I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z. Don't give me the solution. Ask me questions that help me figure it out, and point me to the relevant files or line ranges only if I ask.
```

Use the stories like tickets from a tech lead: read the relevant files, make the smallest safe change, test the workflow, then explain what changed.

