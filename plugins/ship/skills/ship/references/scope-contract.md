# Scope contract & the standard (phases 1 and 4)

## Why size is a lever

Review load on one production monorepo's recent PRs, plotted against added lines:

| Added lines | Files | Comments drawn |
|---|---|---|
| 1–100 | 1–3 | 2–4 |
| 100–500 | 2–10 | 3–6 |
| 600–2600 | 10–26 | 7–9 |

Two or three of those comments are the bots' baseline — they will say something
about anything. The *marginal* load roughly triples past ~300 added lines, because
a large diff defeats the reviewer's ability to hold the whole thing in mind, which
is precisely the faculty that catches blast-radius bugs. A big PR gets a worse
review *and* more rounds.

## The contract

Four lines, in chat, after EXPLORE and ASK (SKILL.md 0a–0b) and before planning:

```
Behavior:     <one observable change, one sentence>
Out of scope: <everything noticed and deliberately not done>
Budget:       <est. added lines / files>   [split: PR1 …, PR2 …]
Base:         <BASE> | <branch this stacks on>
```

Rules:

- **"and" means two PRs.** "Fix the leak and add a metric" is a stack, not a PR.
- **Out of scope is not optional.** Writing down the thing you noticed is what keeps
  it out of the diff. It also becomes the follow-up ticket for free.
- **Over budget → declare the split now.** Bottom-up, each PR independently
  reviewable and independently revertable. Stack it with the repo's stacking tool
  (`gh stack`, Graphite) or plain dependent branches, and merge the stack with that
  tool rather than PR by PR.
- **A migration is always its own bottom PR**, carrying any generated schema types
  regenerated on the same branch. Code that reads the new columns stacks on top.
  The repo's own migration rules (labels, isolated preview databases, naming) come
  from its instruction files and the overlay's `Scope` section.
- **A newly public route** ships with whatever the repo keeps in step with its
  public API — a CLI command, an SDK method, the OpenAPI spec, a version bump — in
  the same PR.

## What legitimately makes a PR big

Not everything can be 300 lines. These are fine, and worth saying so in the
description so the reviewer knows where to spend attention:

- Generated or mechanical bulk — types, fixtures, a rename across call sites.
- Tests. Test volume is not review load; it is the opposite.
- A new module that is genuinely one behavior and has no smaller shippable slice.

What is *not* fine: a fix plus an unrelated cleanup, two behaviors that happen to
touch the same file, or refactoring on the way past. Those are the diffs where the
reviewer's attention runs out before the real bug.

---

## The standard

<a id="the-standard"></a>

### YAGNI
Build what is in the contract. No config flag "for later", no extra parameter with
a default, no abstraction with one implementation. If you find yourself adding a
seam for a future case, that case is a future PR.

### KISS
- Early returns over nesting; no nested ternaries.
- In TypeScript: no `any`, no `unknown` outside a parse boundary, no `@ts-ignore`,
  no `!`.
- A comment explaining *what* the code does means the code needs a better name. A
  comment explaining *why* is valuable — keep those.
- Prefer the boring construct that matches the file it lives in.

### DRY
Extract at the third occurrence, not the second. Extract *into the layer the repo
already has* — a repository method, an existing service, a shared-package helper
— rather than inventing a new `utils/` file. One logical unit per file, named after
it (`my-function.ts`, `MyComponent.tsx`).

### Where classes belong
Style guides often say "avoid classes" while the codebase is full of them
(`TokenService`, `UserRepository`). Both are right, at different altitudes:

- **Classes** for the long-lived shared layers that hold dependencies and group
  cohesive operations: repositories, services, clients. Where that is the
  established pattern, follow it rather than inventing a functional alternative
  alongside it.
- **Functions** for everything else: route handlers, transforms, calculations,
  React components, hooks. A class with no state and one method is a function with
  extra syntax.

"Avoid classes" means *do not reach for a class to hold a pure calculation* — not
*do not have service classes*.

### Testable by construction
Separate the decision from the effect: a pure function that computes what should
happen, and a thin caller that does it. Then the interleavings and provider shapes
from phase 2 are testable without mocking the world — which is what makes the
phase-5 findings pinnable.

Tests come **after** the implementation. Cover:

- the interleavings named in phase 2 (hold a promise pending; assert ordering),
- one real payload per provider for any parser or classifier,
- the exit paths — unmount, close-and-reopen, cancel-during-work,
- error cases, not just the happy path.

Never a fixture derived from your own implementation. See `hunt.md` pattern 9.
