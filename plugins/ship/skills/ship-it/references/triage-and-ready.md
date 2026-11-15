# Stages D & F — triage, fix, ready

Stage C returns a state file, not a verdict. Nothing in it is authoritative: a
fresh reviewer with no context produces real findings *and* confident nonsense,
and telling them apart is the author's job. That is what this stage is.

## 1 · Read only what you need

`$SK` is pr-review's scripts directory, found as in `review-dispatch.md` §0.

```bash
"$SK/state.py" get --owner <OWNER> --repo <REPO> --pr "$PR" | jq '{lastRound: .lastRound, counts, openBySeverity, open}'
"$SK/state.py" show --owner <OWNER> --repo <REPO> --pr "$PR" --id '<one finding id>'   # proof/comment, only when you need it
```

`get` returns the folded open set — summary fields only. Never `cat` the state
file into context: a `max` round carries every finding's proof, comment and
explanation, and you need the proof only for the ones you are about to act on.

## 2 · Print each finding, then gate

Before the `AskUserQuestion` call, print each candidate in chat as its own block:
severity, `file:line`, the claim, the `proof`, and the `chatExplanation` — the
plain-English one, because the point of the gate is deciding with understanding
rather than triaging labels.

Then gate: **`AskUserQuestion`, `multiSelect: true`**, severity-ordered, paged in
fours (the widget caps at four options per question; page 1 carries the highest
severities). Question text names the recommended set — every Blocker and High,
unless you have a written reason one is wrong.

Each option is one finding, labelled `<severity> <file>:<line> — <short claim>`,
its description carrying the impact in one line. For any finding whose fix would
blow the phase-1 budget, add "fix in a follow-up PR" as its own option rather than
forcing fix-or-waive.

**A finding you believe is wrong still goes on the list** — as an option whose
description says why you think it is wrong. Silently dropping it is how the one
that was actually right gets lost.

If a page's answer does not come back, re-ask that page by itself. Do not read
silence as a decline, and do not fix anything that was not selected.

## 3 · Fix

Same standard as `ship` phase 4, plus one rule that matters more here:

- **The smallest change that closes the finding.** A review finding is the most
  common way a 200-line PR becomes a 600-line one, because the fix arrives with a
  refactor attached. If the honest fix is large, that is a follow-up PR, not a
  wider diff.
- Each fix gets the test that pins it, written after the fix.
- Re-read the blast-radius table from `ship` phase 2 before touching shared state:
  a fix is a change, and it has its own siblings and interleavings.

The critic's `gap` list (if stage C produced one) is not a finding list — each gap
is an area no lens opened. Spot-check each one yourself and either write why it is
fine or open it as a finding.

## 4 · Waive the rest, in the PR body

Every unselected finding becomes a line in the PR description, with its reason:

```
Reviewed and not addressed:
- Medium · connect-session.route.ts:88 — retry has no backoff. Deliberate: the
  gateway already rate-limits, and a backoff here delays the failure signal.
```

This is the cheapest round you will ever save. A reviewer who finds the same thing
reads the answer instead of writing the comment. It also merges with the waivers
`ship` phase 5 already put there — one list, not two.

## 5 · Re-verify

`ship` phase 6, scoped to what the fixes touched, output quoted. A check that ran
nothing is not a pass — read the output, do not trust the exit code. Run the
repo's full check once here, before the PR goes ready.

## 6 · Re-review, only when it is earned

Dispatch `/pr-review <N>` in a **fresh subagent** (same pattern as stage C —
it finds this round in the state file, sees the head has moved, and reviews only
the new commits) when either holds:

- a **Blocker or High** was fixed — the fix is new, unreviewed code in exactly the
  place the reviewer already found something, or
- the fixes together add more than ~50 lines, touch a file the original diff did
  not, or cross into another subsystem.

Otherwise skip it and say so. A second full review of three renamed variables buys
nothing.

## 7 · Stage F — ready (after stage E's live tests)

`gh pr ready <N>` invites people to spend their time, so it is gated on an explicit
answer, never on "CI is green" or "looks good to me". One `AskUserQuestion` carrying
both halves of the decision:

- **Single-select: mark ready?** — recommended option first (`Mark ready — 2 Highs
  fixed, 1 Medium waived`), then "hold as draft, I want to look first", then "hold —
  more work needed".
- **Single-select: start the PR watcher?** — stage G's consent question, worded
  as `references/watch.md` §1 gives it. It is asked whichever way the first
  question goes.
- Show the **final PR body** in chat immediately before the call: the one-sentence
  behavior, the issue reference, accepted widening in the reviewer's terms, sibling
  coverage, the **Tested** section (one line per T, with evidence), and the
  combined waiver list. Body edits happen before the flip, not
  after.

After the flip, and after the watcher spawn if one was chosen, report: PR URL,
review depth that ran, findings fixed vs waived, whether a re-review round ran,
and the watcher's session link (or "no watcher"). That is the handover — no
summary of the implementation, which the PR body already carries.
