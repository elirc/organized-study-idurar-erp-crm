# Code Review Mindset

## The review layers (in order)

Review top-down; don't nitpick style before you've checked correctness.

1. **Does it work?** Run it / read it — does it do what the PR says?
2. **Is it correct?** Edge cases, error paths, the hostile input, the concurrent request.
3. **Will it stay correct?** Does it break an invariant, a contract, or a convention that future changes rely on?
4. **Does it fit?** Does it follow the codebase's patterns (the factory, the envelope, soft-delete, server-authoritative math) or fight them?
5. **Is it kind to the next maintainer?** Readable, named well, no surprises, no dead code.

Most junior reviews stop at layer 1 or jump to layer 5 (style). The mid-level signal is spending your attention on layers 2–4.

## Repo-specific review checklist

Apply to any PR against IDURAR:

**Correctness / data integrity**
- [ ] Money math uses `calculate` (`helpers.js:47`), never raw `+`/`-`.
- [ ] Multi-document writes: is the invariant maintained? Is atomicity needed (payment flow)?
- [ ] Does a query touch a soft-deletable model without `removed:false`?
- [ ] Are totals/ownership recomputed server-side, not trusted from the client?

**Security**
- [ ] New route: authenticated mount or intentionally public?
- [ ] `req.body` spread into a write (mass assignment)?
- [ ] User input into a regex/query without escaping/guarding?
- [ ] Error responses not leaking internals in prod?

**Fit / convention**
- [ ] Uses the CRUD factory + override pattern rather than a bespoke route?
- [ ] Returns the `{success, result, message}` envelope with a status that matches?
- [ ] Frontend calls go through `request/request.js`, not raw axios?
- [ ] React: effect deps correct? Derived state not duplicated in `useState`?

**Maintainability**
- [ ] Dead code / commented blocks removed (the repo has a lot already)?
- [ ] Names match the domain glossary?
- [ ] Given no tests exist, is there a manual verification note — or, better, a new test?

## How to write a review comment

The formula: **observation → why it matters → suggestion → openness.**

Bad (vague, unkind):
> This is wrong.

Bad (nitpicky, misses the point):
> Missing semicolon on line 12.

Good (specific, kind, actionable):
> This spreads the whole `req.body` into the update (`update.js:9`), which lets a client set fields like `credit` or `removed`. Could we whitelist the fields we actually expect here? Happy to pair if the field list is fuzzy.

Note what the good comment does: cites the line, names the concrete risk (not "insecure"), proposes a fix, and offers help. That tone gets bugs fixed; a harsh or vague tone gets ignored.

## Blocking vs important vs optional

Label your findings so the author knows what must change:
- **Blocking:** correctness, security, or data-integrity — must fix before merge (e.g. the mass-assignment example).
- **Important:** should fix, but could be a fast-follow (e.g. missing index).
- **Optional / nit:** style, naming, preference — clearly marked so it's not confused with a blocker.

Conflating a nit with a blocker wastes everyone's time; not flagging a blocker as blocking lets bugs ship.

## What NOT to do

- Don't rewrite the PR in comments — suggest, don't dictate.
- Don't bikeshed style the linter could enforce.
- Don't approve to be nice — an approved bug is your bug too.
- Don't demand tests the repo's infrastructure can't run yet — offer to help add the harness instead.

## Interview angle

"Walk me through how you review a PR" and "tell me about a review where you caught something important." Use the layered method and one concrete example from the review katas ([04-code-reading-gym/04-review-katas.md](../04-code-reading-gym/04-review-katas.md)) — e.g. catching the credential leak in Kata 7 or the N+1 in Kata 2. See [08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md) Story 3.

## Drill

Take review Kata 4 (mass assignment) from the reading gym. Write the full review: one blocking comment, one important, one optional, each in the observation→why→suggestion→openness format. Self-grade — Strong: your blocking comment is specific and kind, and you didn't mislabel the nit as blocking.
