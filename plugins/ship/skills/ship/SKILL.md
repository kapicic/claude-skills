---
name: ship
description: Take a change from ticket to merge-ready PR in one review round. Use when implementing a feature, fix, or refactor that will become a pull request — "implement X", "fix Y", "let's build Z", an issue key or URL, or /ship. Runs a fixed pipeline — explore the code, ask (hard stop for answers), scope contract, blast-radius map, plan (hard stop, per-decision approval), build, adversarial self-hunt on the diff (hard stop on findings), local verification with quoted output, PR — aimed at the findings that actually cost review rounds: state whose contract is wider than the diff, sibling call sites the fix missed, and interleavings across an await. Supersedes superpowers:brainstorming and superpowers:test-driven-development — do not invoke either; tests come after the implementation, never before.
---

# Ship

One behavior, mapped before it is written, hunted before it is pushed.

## Why this shape

Review comments on 17 PRs in one production TypeScript monorepo, by author:

| Source | Comments | What they found |
|---|---|---|
| Human reviewers | 19 | **100% state-ownership, sibling-callsite, or interleaving bugs.** Zero style, zero types, zero missing tests-as-such. |
| Review bot (CodeRabbit) | 68 | Top categories: Data Integrity & Integration (14), Stability & Availability (9). Same class. |

And review load tracks size almost linearly: PRs under ~200 added lines drew ~3 comments; PRs over ~600 drew 7–9.

So the two levers are **blast radius** (phase 2 + 5) and **size** (phase 1). Everything else in this skill exists to keep those two honest.

A third thing decides how expensive a correction is: **when the human's input lands.** A fork answered in ASK (phase 0b) costs one answer to change; the same fork first seen inside an approved plan costs a redo. So the code is read and the questions are asked before any scope contract or plan is written.

## The pipeline

Run the phases in order. Do not merge phases. Do not skip ahead because the change "looks small" — three of the real findings below came out of diffs under 30 lines.

| # | Phase | Gate |
|---|---|---|
| 0a | EXPLORE — read the code, name the unknowns and forks | — |
| 0b | ASK — questions built from what exploration found | **HARD STOP: the human's answers** |
| 1 | SCOPE — one behavior, budget, split plan, written from 0a–0b | — |
| 2 | MAP — blast radius table | — |
| 3 | PLAN — decisions list + 1 + 2 + approach | **HARD STOP: which decisions to change (none = approve)** |
| 4 | BUILD — implement, then tests | — |
| 5 | HUNT — adversarial self-review of the diff | **HARD STOP: no open finding** |
| 6 | VERIFY — the repo's checks, output quoted | — |
| 7 | PR — commit, push, open | — |

---

## Repo rules: instruction files and the overlay

This skill carries the method. The repo carries its own rules: its base branch,
check commands, migration conventions, and the docs that say what each layer must
do. Two sources, read before phase 0a:

1. **The repo's instruction files** — `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md` —
   and the rules, invariant or architecture docs they link for the paths you touch.
2. **Your private overlay for this repo**, if you keep one:

   ```bash
   O=~/.claude/ship/$(gh repo view --json nameWithOwner -q .nameWithOwner | tr / -).md
   [ -f "$O" ] && cat "$O"
   ```

   Its `##` sections are named after phases: `Base`, `Explore`, `Scope`, `Map`,
   `Hunt`, `Build`, `Verify`, `PR` (and `Ticket`, `Test plan`, `Watch`, `Watch locally` for `ship-it`). Each
   section adds to the matching phase below, and wins where the two disagree. The
   overlay is for what the repo's own files do not say — your team's labels,
   commands, and the traps you have already hit.

`<BASE>` below is the branch this PR targets: what the instruction files or the
overlay's `Base` section say, else the remote's default branch
(`git symbolic-ref --short refs/remotes/origin/HEAD | sed 's#^origin/##'`, else
`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`) — or, in a
stack, the branch below this one.

## Every choice goes through AskUserQuestion

Whenever this pipeline needs a decision from the human — an open design decision, a split shape, plan
sign-off, which findings to fix, how to land the PR — **ask with the
`AskUserQuestion` tool, never as prose in chat.**

- **Default to `multiSelect: true`.** Use single-select only when the options are
  genuinely mutually exclusive (you cannot both approve the plan and re-scope it).
  Anything that is a *set* — findings to fix, packages to verify, follow-ups to
  file — is multi-select.
- Put the recommended option first and mark it `(Recommended)`.
- Options are concrete, not abstract: name the finding, the file, the split. Use
  `preview` when the choice is between two shapes the human should see side by
  side (a split plan, two implementations).
- One call per gate, batching the questions that gate together — not one call per
  question.

The phases below name the specific question each gate asks.

## 0a · EXPLORE

Read before you decide anything. Read-only: no edits, no scope contract, no plan yet.

Read, in parallel where the reads are independent:

- **The area the change touches** — the files, their callers and callees, and the current behavior (run or trace it when that is cheap).
- **Sibling implementations of the same shape** — the greps in `references/blast-radius.md`.
- **The repo's rules for every path you expect to touch** — the instruction files, and the layer rules, invariants or architecture notes they link (repositories, pagination, outbound fetches, metrics, migrations…), plus the overlay's `Explore` section.
- **The tests around it** (what pins today's behavior) and `git log` of the area (recent changes, prior fixes).
- **The input** — the ticket or request and anything it links, and open PRs touching the same files.

Time-box it: explore until you can name the forks and the unknowns, not until you have read everything.

Report in chat — facts, not a plan:

- **Found:** 3–8 bullets, each with `file:line`.
- **Unknowns:** what the code cannot tell you — intent, constraints, rollout, who depends on it, which of two reasonable behaviors is wanted.
- **Forks:** every place the approach could reasonably go two ways, and what each way costs.

## 0b · ASK — hard stop

Turn the Unknowns and Forks into questions. `AskUserQuestion`, one call per round:

- One question per **open decision**: approach forks (with a `preview` of each shape), scope boundaries (what is in, what is deliberately out), and anything only the human knows. Each option is concrete and names its cost; the recommended one goes first.
- **Every open decision, and nothing the code already answers.** Not "does this look right?" — a question with options.
- Offer "not sure — look at <X> first" on a fork you could not settle by reading; picking it sends you back to 0a for that one thing.
- After each round, re-check: did an answer open a new fork? Ask again until Unknowns and Forks are empty. Do not start SCOPE with one open.
- If exploration left nothing open, say so in one line — the files you read and why each decision is settled — and continue. That is the only way this phase is skipped.

The answers go into the scope contract verbatim, and they are the source of the plan's decisions list.

## 1 · SCOPE

Write a scope contract from the EXPLORE findings and the ASK answers, not from the prompt alone. Four lines, in chat:

- **Behavior:** one sentence, one observable change. If it needs "and", you have two PRs.
- **Out of scope:** everything you noticed and are deliberately not doing. Naming it here is what stops it leaking into the diff.
- **Budget:** target ≤300 added lines, ≤8 files. If the change cannot fit, say so now and state the split as a stack — bottom PR first.
  **Over budget → ask.** `AskUserQuestion`, single-select, each option a concrete
  split with a `preview` showing the stack (`PR1 migration+types → PR2 repo+route
  → PR3 UI`). Include "land it as one PR" as an option, with the comment cost
  named.
- **Base:** `<BASE>`, or the branch this stacks on.

Then, before planning:

- Confirm EXPLORE covered the repo's rules for every path in the contract; read whatever it missed.
- If a migration is involved: it goes in its own bottom PR, with any generated schema types regenerated on the same branch. Code that reads the new columns stacks on top. The overlay's `Scope` section adds the repo's own conditions (labels, isolated databases, version bumps).

Full rules, including when to split and how to stack: `references/scope-contract.md`.

## 2 · MAP

Before writing code, build the blast-radius table. This is the phase that pays for the skill.

For every piece of mutable state the change touches — a DB column, a row status, a React ref, a module-level variable, a cache entry, a lease — fill in:

| State | Who owns it | Other writers | Other readers who gate on it | Concurrent access |
|---|---|---|---|---|

Then answer three questions in writing. Each one maps to a real finding on a real PR:

1. **Widening** — does the field/flag/status I am setting already mean something to code I am not touching?
   *(`disabled_at` set for a scope-only fault also skipped every scheduled job on that account, and accrued toward auto-pause.)*
2. **Siblings** — how many implementations of this shape exist, and does my change cover each one?
   *(A 403 classifier built from Gmail's `errors[]` payload silently missed One Platform's `details[]` and Graph's `UpstreamHttpError` entirely.)*
3. **Interleaving** — for each `await` in the path, what can happen while it is pending?
   *(`releaseSession()` returned before teardown finished, so retry started a new browser into the old one's concurrency slot.)*

Method and the exact greps: `references/blast-radius.md`.

## 3 · PLAN — hard stop

Present, in chat: the scope contract, the blast-radius table, the three answers, and the implementation approach (files, new units, what gets tested) — with **every decision on its own line**:

```
D1  <choice> — <why>; rejected: <alternative>
D2  <choice> — settled in ASK
```

Nothing in the plan may be a choice the human was not already asked about. If writing the plan surfaces a new fork, go back to 0b and ask it; do not bury it in the approach.

**Stop. Do not write code until the human signs off.** Ask with
`AskUserQuestion` — do not ask in prose and do not proceed on silence:

- *"Which decisions do you want to change?"* — **`multiSelect: true`**, one option per decision (`D1 <choice>`, the alternative in the description), plus **"None — approve the plan"** first and marked `(Recommended)`, and **"Re-scope the PR"** last. Unselected decisions are accepted. If "None" is picked together with decisions, the decisions win.
- Where phase 2 surfaced accepted widening or an uncovered sibling, ask that in
  the same call as a second **multi-select**: *"Which of these does this PR take on?"*,
  one option per item, so the unselected ones become documented limitations
  rather than silent ones.

**A change is a one-decision edit.** For each decision picked, ask one follow-up (`AskUserQuestion`, options are the alternatives, `preview` when the shapes differ), revise only those lines, and re-present the changed lines plus what they touch — the blast-radius rows and the budget. Repeat until the answer is "None". Editing a decision never restarts the pipeline; only "Re-scope the PR", or a change that breaks the scope contract, goes back to phase 1.

If the map changed the scope, re-scope — never quietly widen the PR.

## 4 · BUILD

Implement, then test. Not test-first.

- YAGNI — build the behavior in the contract, nothing adjacent. No config knob, no abstraction layer, no "while I'm here".
- KISS — early returns, no nested ternaries, no clever generic. If it needs a comment to say *what* it does, rename it instead.
- DRY at the third occurrence, not the second. Extract into the layer the repo already uses (a repository method, a shared service class, a shared-package helper) rather than a new one-off util.
- Match the surrounding file's idiom, naming and comment density.
- Tests after: cover the interleavings and sibling shapes from phase 2 — that is the whole point of having mapped them. Never a fixture hand-built in the shape the code already expects (see pattern 9 in the hunt reference).

Standards, and where classes belong: `references/scope-contract.md#the-standard`.

## 5 · HUNT — hard stop

Read your own diff cold, as a reviewer who did not write it:

```
git --no-pager diff <BASE>...HEAD
```

Then walk `references/hunt.md` (plus the overlay's `Hunt` section) — 17 patterns, every one taken from an actual finding on a real PR (1–12 from human reviewers, 13–17 from bot findings that were accepted). For each pattern, either write why it cannot apply, or write the finding.

A finding is only real if you can write the concrete sequence. **"Consider concurrency" is not a finding. "A calls X, awaits; B sets status=failed; A resumes and writes status=running" is.** Use the reviewer format:

```
Bug: <one line>
<mechanism, with file:line for every claim>
Impact: <what a user or operator sees>
Fix: <smallest change that closes it, plus the test that pins it>
```

**No PR while a finding is open.** Triage with `AskUserQuestion`,
**`multiSelect: true`** — one option per finding, labelled with the pattern and
the file (`#4 lease has no owner — complete-job.route.ts:198`), the
description carrying the impact in one line:

> *"Which findings do I fix in this PR?"* — selected = fixed now; unselected =
> waived, and the waiver goes in the PR description with its reason.

If a finding is large enough to blow the budget, offer "fix in a follow-up PR" as
its own option rather than forcing fix-or-waive.

## 6 · VERIFY

Run the checks the repo's instruction files name — and the overlay's `Verify`
section — from the directory they say, and quote real output. Nothing here is
optional and nothing is assumed to pass.

- Cheapest first, stopping at the level that covers the change: format check, lint on the changed files, the affected tests, then typecheck or build for the touched packages.
- No instructions? Use what the repo defines — `package.json` scripts, a `Makefile`, `justfile`, `tox.ini`, the CI workflow's own steps — scoped to the touched packages.
- **A check that ran nothing is not a pass.** A workspace filter that matched no package, a test run that collected zero tests, a glob that matched no file — these often exit 0. Read the output; a silent exit 0 is a false pass, not a green run.
- If a pre-commit hook runs format and lint, skip those two steps here. The phase-7 commit runs them; read the hook's output and fix what it blocks.
- Run the repo's full check once, before opening or updating the PR — not per commit.

If a check fails, fix it and re-run. Report failures with their output; never summarize a red run as "mostly passing".

## 7 · PR

Commit with a conventional message and open the PR against `<BASE>`:

```bash
gh pr create --base <BASE> --title "<type>(<scope>): <behavior>" --body-file <file>
```

Follow the repo's PR template and conventions; the overlay's `PR` section may name a skill or format to use instead. Stacking → the repo's stacking tool (`gh stack`, Graphite) or plain dependent branches, each PR based on the one below; merge a stack with that tool, not PR by PR by hand.

The body must carry the phase-2 conclusions. This is the cheapest round you will ever save: when a reviewer's question is already answered in the description, it does not become a comment. Include:

- the one-sentence behavior,
- any **widening** you accepted, in the reviewer's own terms ("this also stops scheduled jobs on that account; intentional because …"),
- which **siblings** are covered and which are explicitly not,
- any waived finding from phase 5.

Never push to a branch you do not own, and never post to GitHub, without explicit permission for that action.

## Anti-patterns

| Move | Why not |
|---|---|
| Skipping phase 2 because the diff is small | Three of the 19 real findings were in diffs under 30 lines. Size predicts *review load*, not *blast radius*. |
| Hunting for patterns "in principle" | A pattern check with no concrete interleaving written down is a check that did not happen. |
| Fixing a finding by widening the PR | Re-scope instead: land the narrow fix, stack the rest. |
| Invoking `superpowers:brainstorming` or `test-driven-development` | This skill replaces both. Exploration, questions and planning happen in phases 0a–3; tests happen in phase 4, after the code. |
| Asking a gate question in prose | Prose questions get skimmed and answered ambiguously; `AskUserQuestion` gets an answer you can act on. |
| Writing the scope contract from the prompt alone | The code has not been read, so Behavior, Out of scope and Budget are guesses. Explore (0a) and ask (0b) first. |
| First showing the human a choice inside the plan | Approving a bundle is all-or-nothing, and changing one decision feels like a redo. Every fork is a 0b question; the plan only ratifies. |
| Asking "does this look right?" | Not a decision. A question has options, a recommendation and a named cost. |
| Exploring until everything is read | The skill is slow enough. Stop when the forks and unknowns are named. |
| Fanning out subagents for the hunt | The hunt is first-person on purpose. `pr-review` exists if a second opinion is wanted — invoke it deliberately, not as a substitute for phase 5. |
