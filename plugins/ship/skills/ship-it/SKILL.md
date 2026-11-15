---
name: ship-it
description: Full delivery pipeline for a change that becomes a pull request — tracker ticket, adapted ship (whose plan asks where and how to test, built from the feature's surfaces), fresh-context self-review of the draft PR, triage and fix, live tests, then mark ready for review and hand the PR to a cloud auto-fix watcher for CI failures and review comments. Use for "ship it", "/ship-it", "implement X end to end", an issue key or URL handed over for implementation, or any implement/fix/refactor task that should arrive review-ready rather than as a draft someone else has to clean up. Wraps the `ship` skill (which stays usable standalone for a quick ticketless change) and the `pr-review` skill (run by a fresh agent that never saw the build). Supersedes superpowers:brainstorming and superpowers:test-driven-development — do not invoke either.
---

# Ship it

Every PR starts as a ticket and ends as a PR that has already been reviewed once,
by someone who did not write it.

## Why this shape

`ship` is one pass: explore, ask, scope, map, build, hunt, verify, PR. It is a good pass, and it
still ships PRs that draw review rounds, for two structural reasons:

- **No pre-processing.** Work that starts from a chat message has no durable
  statement of intent. The scope contract lives in a transcript nobody reads, and
  the PR is judged against a goal only its author knows.
- **No post-processing.** Phase 5's hunt is first-person by design — that is its
  strength on interleavings and sibling call sites, and its blind spot everywhere
  else. **You cannot catch what you didn't think of by thinking about it harder in
  the same context.** The author's reading of their own diff is anchored on the
  intent, not on the text.

So this pipeline brackets `ship` with the two things it lacks: a ticket in front,
and a genuinely fresh reviewer behind. The reviewer is a separate agent with an
empty context, working from a worktree — it sees the diff the way the human
reviewer will, and its findings get triaged and fixed **before** anyone is asked
to look.

After the human review starts, a cloud watcher takes over the mechanical part of
each round. It fixes red CI and small comments, and it asks you about the rest,
so the PR does not stall while the author's session is closed.

## What it needs

- **`ship`** — the sibling skill in this plugin.
- **`pr-review`** — the `pr-review` plugin (stage C runs its scripts and skill).
  Without it, stop at stage C and say to install it.
- **Optional:** `tracker-updates` for stage A's issue mechanics, and `cloud-pr`'s
  `claude-cloud-spawn` for stage G. Without them, stage A falls back to `gh issue`
  or "skip the ticket", and stage G to a local watcher or none.
- **The repo's rules** — its instruction files and your overlay for this repo, read
  exactly as `ship` describes ("Repo rules"). ship-it also reads the overlay's
  `Ticket`, `Test plan`, `Watch` and `Watch locally` sections.

`<OWNER>/<REPO>` is `gh repo view --json nameWithOwner -q .nameWithOwner`, and
`<BASE>` is resolved as in `ship`.

## The pipeline

| # | Stage | Gate |
|---|---|---|
| A | TICKET — a tracker issue exists (given, or created) | **HARD STOP: confirm before creating** |
| B | SHIP — the `ship` skill, phases 0a–7, with the overrides below | ship's own stops (ASK, PLAN + **where/how to test**, HUNT) |
| C | REVIEW — fresh agent reviews the draft PR at `effort=auto` depth | — (runs headless) |
| D | TRIAGE + FIX — decide per finding, fix, re-verify | **HARD STOP: which findings** |
| E | LIVE TEST — run the post-push half of the test plan | **HARD STOP only if a planned test fails or cannot run** |
| F | READY — `gh pr ready` | **HARD STOP: explicit go-ahead** (same call asks whether to start G) |
| G | WATCH — a cloud session sweeps CI and open threads once, then native auto-fix handles each new CI failure and review comment | consent comes from F's gate; the watcher asks you in its session about anything outside the mechanical split |

Do not merge stages. Do not skip C because the hunt in B "already covered it" —
they are different readers, and C exists precisely because B is not independent.

**Every gate goes through `AskUserQuestion`**, never prose — same rule and same
shape as `ship` (recommended option first and labelled, `multiSelect: true`
unless the options genuinely exclude each other, options naming the concrete
file/finding/split). One call per gate, batching whatever gates together.

---

## A · TICKET

The PR needs a tracker issue. The tracker is the one the overlay's `Ticket`
section names, else whatever the `tracker-updates` skill detects (a pasted link,
its configured tracker, a connected Linear or Jira MCP), else GitHub Issues
through `gh issue`. One of two paths:

- **Issue given** (a key such as `ABC-123`, a tracker URL, or "the ticket about
  X") — fetch it with the tracker's tool, and read its description and comments as
  the input to `ship` phase 0a (EXPLORE). A ticket is a starting point, not a
  decided scope: what it leaves open is what phase 0b asks about.
- **No issue** — run `ship` phases 0a–0b first (explore, then ask); a ticket
  drafted before the code is read describes a guess. Then draft one from the
  settled scope and create it after confirmation. Use the `tracker-updates`
  skill, when installed, for the house style and the tracker's posting mechanics
  (issue shape: why → **Scope** → **Out of scope**). Confirm with
  `AskUserQuestion` — single-select, the drafted title as the recommended option,
  plus "adjust the title/scope" and "skip the ticket for this one". Print the full
  drafted body in chat immediately before the call so the choice is made with the
  text in view.

Nothing else is written to the tracker by this pipeline — no status transitions,
no plan comment, no PR attachment. If stage B's MAP materially changes the scope,
say so in chat and offer an issue-body update; do not make one unasked.

Carry the issue reference forward: it goes in the PR body the way the repo links
issues — a bare key such as `ABC-123` when the tracker autolinks it, `Closes #123`
for GitHub Issues, or what the overlay's `PR` section says — and nowhere else.

## B · SHIP

Invoke the `ship` skill and run its phases (0a–7) as written; if stage A already
ran 0a–0b on the no-issue path, continue at phase 1. Four overrides, and nothing
else changes:

| # | Override | Why |
|---|---|---|
| 1 | **The PR opens as a draft** — `gh pr create` gains `--draft` | Stage C reviews it before a human is invited |
| 2 | **The PR body carries the issue reference** | The reviewer gets the intent |
| 3 | **Phase 5 HUNT still runs in full** | Stage C is additive. A hunt skipped in favour of "the reviewer will catch it" costs a whole extra round trip and loses the one class the author is best at |
| 4 | **Phase 3 PLAN also asks where and how to test**, built from the phase-2 map. The answers become `T` lines beside the `D` lines. Phase 6 runs the pre-push `T` lines as well as its own checks | The right test depends on the feature. What can be reached (a preview, a test account, a real third-party connection) is something only you know. Procedure: `references/test-plan.md` |

Phase 0b (ASK), phase 3 (PLAN) and phase 5 (HUNT) keep their hard stops. Phase 6's verification
output is quoted, not summarized. Do not push to a branch you do not own.

## C · REVIEW — fresh agent, `effort=auto`

The reviewer never sees this conversation. Its depth is `effort=auto`, from
pr-review's own `profile` of the pushed diff: code lines, surfaces, and the risk
flags pr-review is configured with (migrations, auth, public API, …). You may
raise the tier one step, with a named reason; auto never goes past `xhigh`, and
`max` only runs on your explicit override. At `low`/`medium`/`high` a fresh
subagent runs `pr-review` at the resolved tier; at `xhigh` the parent runs the
`Workflow`, with the correctness lens sharded on diffs over 1,500 code lines.

Full procedure — finding pr-review's scripts, sizing, both dispatch paths, the
exact reviewer prompts, the findings contract and the state file:
`references/review-dispatch.md`.

Two invariants for this stage: **nothing is posted to GitHub** (a self-review
comment on your own draft is noise — findings live in chat and in the state file),
and **the parent reads only the findings**, never the reviewer's transcript.

## D · TRIAGE + FIX

Read the state file, print each finding as its own block, then gate with
`AskUserQuestion` (`multiSelect: true`, severity-ordered, paged in fours) —
selected findings get fixed, unselected ones become waivers written into the PR
body with their reason. Fix, re-verify with `ship` phase 6, and re-review only
when the fixes were substantial enough to warrant it.

Procedure, including the re-review trigger: `references/triage-and-ready.md`.

## E · LIVE TEST

Run every `[post-push]` T line from the plan — against the draft PR's preview when
the repo deploys one, or wherever the T line says. This stage comes after triage,
so the tests exercise the code that will ship. Record each T line as ran, failed
or could not run, with its evidence. A failure is a finding: fix it, re-verify,
and re-run the T lines the fix invalidates. A T line that could not run goes to
`AskUserQuestion`: hold, swap in another environment, or waive with a reason. If
the plan had no post-push lines, say so in one line and continue.

Procedure and guardrails: `references/test-plan.md` §3–4.

## F · READY

`gh pr ready <N>` is the moment the PR becomes other people's problem, so it needs
an explicit go-ahead — `AskUserQuestion`, never inferred from "looks good". The
same call confirms the final body (behavior, accepted widening, sibling coverage,
the **Tested** section, and waived findings from both the hunt and the review).
It also asks whether to start the stage G watcher. That answer is the per-PR
consent for the watcher's pushes and thread replies.

## G · WATCH — cloud auto-fix

If the F gate said start, spawn a cloud session with `cloud-pr`'s
`claude-cloud-spawn`. It sweeps the current failing checks and unresolved threads
once, then turns on Claude Code's native auto-fix. Auto-fix reacts to the PR's
GitHub events, so nothing polls and no terminal needs to stay open. Report the
session link with the commands to steer and stop it, then end. Do not wait for its
first event.

The watcher fixes mechanical items, pushes them as fast-forwards to the PR branch,
and replies in the threads it acted on. When the PR is not stacked, it merges the
base on a conflict. It asks you in its session about logic, design, security,
scope and anything else, and pings you on Slack once per new question, because
nothing else tells you it is waiting. It never resolves threads, never posts
top-level comments, never changes draft status and never merges. After G starts,
the branch has two writers, so pull before any local push.

Procedure, the full watcher prompt and the fallback when a spawn fails:
`references/watch.md`.

## Anti-patterns

| Move | Why not |
|---|---|
| Starting to code before the ticket exists | The ticket is the only durable statement of intent. Written afterwards it is a description of the diff, which is worth nothing. |
| Letting stage C stand in for the phase-5 hunt | They catch different classes. Dropping the hunt trades the one review the author is uniquely good at for a slower one. |
| Reviewing in this context "to save a dispatch" | The value is the empty context. An agent that watched the code get written reviews the intent, not the text. |
| Auto-fixing every finding without the triage gate | Some findings are wrong, and some are right but out of scope. Fixing them all is how a 200-line PR becomes a 600-line one. |
| Posting the self-review to GitHub | Nobody asked for it, and it buries the human reviewer's own comments under your notes to yourself. |
| Marking ready because CI is green | Green CI is not the gate. The gate is the human saying go. |
| Offering generic test options ("add tests", "manual QA") | Anything is a yes to a generic option. Name the scenario, the file and the environment, taken from this feature's map. |
| Skipping a planned test that could not run | An untested line with no mention reads as tested. Ask, then waive it in the PR body or hold. |
| Starting the watcher without the F-gate answer | It pushes and replies under the user's account. That needs consent for each PR, and the consent is never inferred. |
| Pushing locally after G without pulling first | The watcher may have pushed in the meantime. A rejected push tempts a force push, which deletes its fix. |
| Using the built-in `/autofix-pr` for G | It targets the branch of the directory the session started in — often the main checkout, not the worktree that holds the PR. G names the PR explicitly. |
| Calling a watched PR clean from its threads and CI alone | Review findings outside the diff have no thread. The watcher lists them as waiting on you in its own session. Read its last result before recommending a merge. |
| Polling the watcher from this session | It runs on GitHub events and reports in its own session. Polling spends tokens to learn what that session already shows. |
