# Maintainer Communication

How to work with maintainers and teammates — asking for help without outsourcing your thinking, reporting bugs well, and disagreeing respectfully. These skills are graded in behavioral rounds and in your first months on any team.

## Asking for help without outsourcing your thinking

The difference between a junior and a mid-level ask:

**Junior (outsources thinking):**
> The payment thing isn't working, can you help?

**Mid-level (shows work, asks a precise question):**
> Recording a payment updates the payment doc but `invoice.credit` sometimes doesn't match. I traced it to the two writes in `paymentController/create.js:41` and `:66` not being in a transaction, so a failure between them leaves the invariant broken. I want to add `withTransaction`, but that needs a replica set — is the dev/prod Mongo a replica set, or should I go with derive-on-read instead?

The mid-level version: states the symptom, shows the investigation with anchors, proposes a direction, and asks the *one* decision they can't resolve alone. It respects the maintainer's time and demonstrates you did the work. Use this template:

```
Symptom: <what you observe>
What I've tried / found: <investigation, with file anchors>
My hypothesis: <specific guess>
The decision I'm stuck on: <the one question only you can answer>
```

## Reporting a bug (repro + minimal example)

A good bug report lets the maintainer reproduce in under a minute:

```
## Summary
<one line>

## Steps to reproduce
1. ...
2. ...

## Expected
<what should happen>

## Actual
<what happens, with the error message / status>

## Environment
<node version, branch, anything relevant>

## Suspected cause (optional)
<if you investigated: the file:line and why>
```

**Example for this repo:**

> **Summary:** App shows a white screen on load.
> **Steps:** `npm run dev` in `frontend/`, open `http://localhost:3000`.
> **Expected:** login page.
> **Actual:** blank page; console: `ReferenceError: Quote is not defined`.
> **Suspected cause:** `routes.jsx:73` renders `<Quote/>` but no import exists (`:5-26` has no `Quote` lazy import). Same for `PaymentMode`/`Taxes`.

That last line turns a report into a near-fix and builds your credibility.

## Proposing a feature

Don't open with a solution — open with the problem and let the maintainer shape it:

> **Problem:** the invoice summary endpoint ignores its `type` param, so the dashboard's "this month" total is actually all-time (`invoiceController/summary.js:37-40`).
> **Impact:** the dashboard number is wrong for any period.
> **Proposed direction:** restore the date `$match`; happy to do it. Before I do — is the all-time total intentional anywhere, or is this a straightforward bug?

## Disagreeing respectfully

You'll disagree with maintainers and reviewers. The skill is disagreeing on substance without making it personal or digging in.

**The technique:** acknowledge their point, state your concern with evidence, propose a path, and yield gracefully if overruled.

> I see why hard delete is appealing for storage. My concern is that everything filters `removed:false` and payments reference invoices (`Payment.invoice`), so hard delete would orphan those refs and lose the audit trail. Could we do a scheduled purge of rows removed >90 days instead? If storage is urgent and that's too slow, I'll defer — your call on the priority.

What this does: validates their goal (storage), states the concrete risk (orphaned refs, with an anchor), offers an alternative, and explicitly yields the decision. You can be right *and* easy to work with.

**When you're overruled:** disagree-and-commit. Once the decision is made, support it — don't relitigate in every subsequent comment. If you think it's a serious mistake, say so once, clearly, in writing, then move on.

## Responding to review on your own PR

- Thank the reviewer for catching real issues — genuinely.
- For each comment: fix it, or explain why not (with reasoning, not defensiveness).
- Don't take it personally — the review is about the code, not you.
- If a comment is wrong, push back politely with evidence, the same way you'd want them to.

## Templates summary

| Situation | Open with |
| --- | --- |
| Asking for help | symptom + investigation + the one decision |
| Reporting a bug | repro steps + expected/actual + suspected cause |
| Proposing a feature | the problem, not the solution |
| Disagreeing | acknowledge → concern with evidence → alternative → yield |
| Responding to review | thanks → fix-or-explain → don't relitigate |

## Interview angle

"Tell me about a time you disagreed with a teammate" (STAR Story 3) and "how do you ask for help?" are common. The strong answer shows you did your own investigation first and disagreed on substance while staying collaborative. See [08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md).

## Drill

Write the "asking for help" message for a bug you'd genuinely be stuck on in this repo (suggest: the ReDoS in search — you're unsure whether to escape or switch to `$text`). Use the four-line template. Self-grade — Strong: you show investigation with an anchor and ask exactly one decision, not "what do I do?"
