# Stage G — the cloud PR watcher

After READY, the review round happens on GitHub, often hours later, when the
author's session is gone. Most of that round is mechanical: a red lint job, a
review-bot nit, a rename. Stage G hands those to a cloud session running
Claude Code's native **auto-fix**. Auto-fix subscribes to the PR's GitHub
events (CI failures, review comments) and runs with no polling and no open
terminal. Anything outside the mechanical set comes back to the user as a
question in that session.

Requirements: the Claude GitHub App installed on the repo (auto-fix acts
through it), and `cloud-pr`'s `claude-cloud-spawn` script.

## 1 · Consent — inside the F gate

Stage F's `AskUserQuestion` call carries a second question beside "mark ready?":

- **Start the PR watcher?** Single-select. The recommended option comes first:
  `Start the cloud watcher`, described as "pushes mechanical fixes to
  `<BRANCH>`, replies to review threads under your account (labelled Claude
  Code), merges `<BASE>` on conflict when the PR is not stacked; asks you in the
  session about anything else". The other option is `Not now`.

It is offered whether the PR goes ready or stays a draft. On a draft, the
watcher still handles CI. Some review bots skip drafts, so their comments may
start only after the flip. Spawn **after** `gh pr ready` when both are chosen,
so the watcher's first sweep sees the ready state.

The consent is per PR. "Not now" means no watcher. Do not offer it again later
in the session unless the user asks.

## 2 · Spawn

Take `<N>` from the `gh pr create` output in stage B. **Never work out the PR
from the current directory**, whether with a bare `gh pr view` or the built-in
`/autofix-pr`, which reads the directory the session started in. Sessions often
start in the main checkout and build in a worktree, so that directory's branch is
usually not the PR's branch. Every command below names the PR explicitly, so it
works from any directory inside the repo.

```bash
gh pr view <N> --repo <OWNER>/<REPO> --json number,url,headRefName,baseRefName,title
# the worktree that holds the PR branch (empty if none; then pull is not needed)
WT=$(git worktree list --porcelain \
  | awk -v b="refs/heads/<BRANCH>" '$1=="worktree"{w=$2} $1=="branch" && $2==b{print w}')
# cloud-pr's spawn script: on PATH, else the newest plugin-cache copy
SPAWN=$(command -v claude-cloud-spawn || ls -td ~/.claude/plugins/cache/*/cloud-pr/*/scripts/claude-cloud-spawn 2>/dev/null | head -1)
prompt=$(cat <<'PROMPT'
<the watcher prompt below, with every <PLACEHOLDER> substituted>
PROMPT
)
"$SPAWN" "$prompt"
```

`<BRANCH>` is `headRefName` and `<URL>` is `url` from the `gh pr view` output;
`<BASE>` is `baseRefName`.

No `$SPAWN` → skip to §4 without the retry option, and say `cloud-pr` is not
installed (`/plugin install cloud-pr@kapicic`).

The script prints one `https://claude.ai/code/session_…` link in about 5–10
seconds. It starts from an anchor worktree at the remote's default branch, so the
local checkout is not touched. Its header and the `autofix-pr-cloud` skill in
`cloud-pr` cover the mechanics.

Report this and continue to the handover:

```
Watcher on PR #<N> → <link>
  steer:  claude -p "<message>" --cloud <session_id>
  stop:   claude -p "stop watching the PR" --cloud <session_id>
```

Do not poll the watcher or wait for its first event. Its answers and questions
live in the cloud session.

**Before anyone merges, read the watcher's latest result.** Run `RemoteTrigger`
`get_run_log` with the `session_…` id. Its last `result` line lists everything
still waiting on the user. That includes findings outside the diff, which have
no review thread (some review bots put those in their review body). Resolved
threads and green CI do not mean nothing is open. When it needs a decision, it
pings the user on Slack (§5), and the user answers in that session or with the
steer command.

**From here on the branch has two writers.** Before any local commit is pushed
to `<BRANCH>`, run `git -C "$WT" pull --ff-only` (or merge on divergence). Use
`-C` rather than `cd`, because parallel shells share one working directory.
Never force push over the watcher's commits.

## 3 · The watcher prompt

Substitute `<N>`, `<URL>`, `<BRANCH>`, `<BASE>` and `<OWNER>/<REPO>`. For
`<CONFLICT RULE>`, use the first form when the PR targets the repo's normal PR
base, the second when it is stacked on another PR:

- `` `git merge origin/<BASE>`, resolve, verify, and push. Never rebase onto it and never force push. Ask instead when a resolution needs a product decision. ``
- `` ask, because this PR is part of a stack and restacking is the author's job. ``

For `<REPO RULES>`, paste the overlay's `Watch` section (the cloud session cannot
read your local files), or drop that paragraph when the overlay has none.

> You are the PR watcher for PR #`<N>` (`<URL>`), branch `<BRANCH>`, base
> `<BASE>`, in `<OWNER>/<REPO>`. The author built it with the ship-it pipeline,
> so it was self-reviewed, triaged and live-tested before you started. Your
> job is to keep it merge-ready by handling failing CI and review comments
> within the rules below, until the PR merges or closes. You start with zero
> context.
>
> SETUP. There is no `gh` CLI here. Reach GitHub through `mcp__github__*`. Do
> not call `pull_request_read` with `get_files` or `get_diff`, because either
> exceeds the tool's token cap on a real PR. Get the code with git:
> `git fetch origin <BRANCH> <BASE> && git checkout -B <BRANCH> origin/<BRANCH>`.
> Read the PR body first. Its behavior sentence, issue reference and
> "Reviewed and not addressed" list are the scope contract. A comment that asks
> for something already on that list gets the listed reason as its reply, not a
> fix.
>
> 1. SWEEP once, now. Check runs with conclusion failure, timed_out or
>    action_required (a check that is still running is not a failure), and every
>    unresolved review thread. An outdated position does not mean the thread
>    was fixed, so check the current code. Handle each item as described below.
> 2. Then turn on auto-fix for `<URL>` so new CI failures and review comments
>    reach you as events. Handle each event the same way.
>
> CLASSIFY by content, not by author — bot names change.
> - FIX without asking: lint, format and import order; mechanical type errors in
>   files this PR touches (a missing import, a simple type mismatch, an unused
>   variable); typos, renames, missing null checks or guard clauses, and small,
>   unambiguous suggestions with clear intent.
> - ASK the user in this session, and change nothing until they answer: logic
>   bugs, design or architecture, security, performance trade-offs, questions
>   about requirements or intent, suggestions with several valid approaches,
>   threads with back-and-forth, test-logic failures, type errors that involve a
>   design choice, anything that widens the PR beyond its stated scope or issue,
>   failures in files this PR does not touch, and flaky or infrastructure
>   failures. Report flakes and infra failures with the log line, and do not
>   retry in a loop.
> - NOTHING TO PUSH: a check that needs an approval or a ruleset sign-off, or a
>   duplicate or no-action event. Note it in one line and move on.
>
> REPO RULES for this repo, which add to the split above and win where they
> conflict: <REPO RULES>
>
> FIX STANDARD. Make the smallest change that closes the item, with no refactor
> attached. If the honest fix is large, ask; a follow-up PR is the default
> answer. Before pushing, run the cheapest checks that cover the change, in the
> order the repo's instruction files give (format, lint, affected tests,
> typecheck). Quote their output in the session. If this environment cannot run
> a check, say so. Never claim a check passed when it did not run. Use one commit
> per event: `fix(<scope>): address review feedback` or
> `chore(ci): fix <check> on <BRANCH>`, ending with
> `Co-Authored-By: Claude <noreply@anthropic.com>`.
>
> PUSH only to `<BRANCH>`, and only as a fast-forward. Never force push. The
> author may push too, so re-fetch right before pushing. If the push is
> rejected as non-fast-forward, merge `origin/<BRANCH>`, re-run the checks,
> and retry once. If it is rejected again, stop and report.
>
> CONFLICTS. Auto-fix gets no event when the PR starts conflicting with its
> base, so check `mergeable` during the sweep and after each event. On a
> conflict: <CONFLICT RULE>
>
> GITHUB WRITES. You may reply in a review thread you acted on, once per
> action. Name the fix commit and what changed, or give the reason it was
> skipped, or say it is waiting on the author. You MUST NOT resolve threads,
> post top-level PR comments or reviews, request reviewers, change draft or
> ready status, edit the PR body or labels, merge, close, or push to any
> other branch.
>
> REPORT in this session after each item: what triggered it, what you did (with
> the sha) or why you did nothing, and what is waiting on the user. A one-line
> report is fine. Never invent work to justify an event.
>
> NOTIFY. The user does not watch this session, and GitHub never notifies them
> of your replies, because those post under their own account. When a turn
> leaves something new waiting on the user (an ASK item, a stop-and-report,
> a check that cannot run), send one Slack ping for it before ending the turn.
> Do not ping again for "still waiting" turns, for duplicate events, or for
> fixes you pushed. Send it with:
> `curl -sS -X POST -H 'Content-Type: application/json' --data "$(jq -n --arg text "$MSG" '{text:$text}')" "$SHIP_IT_SLACK_WEBHOOK"`
> where `$MSG` is one line, because the Slack workflow drops newlines:
> `PR #<N> needs your decision: <the question> · Options: <the options> · <link>`,
> with the link from `echo "https://claude.ai/code/${CLAUDE_CODE_REMOTE_SESSION_ID/#cse_/session_}"`.
> If `SHIP_IT_SLACK_WEBHOOK` is unset or the request fails, say so once in
> this session, quoting the HTTP status, and carry on. Never put the webhook
> URL in a commit, a comment or a report.
>
> Stop watching when the PR merges or closes, or when the user says stop.

## 4 · If the spawn fails

The script exits non-zero and prints the tail of the CLI output. The
`autofix-pr-cloud` skill's "If it fails" section covers the known causes
(folder trust on the anchor, an unfetched base branch, the wrong cloud
environment). Show the error, then ask with `AskUserQuestion` (single-select):

- **Retry the spawn** after the fix named in the error.
- **Watch locally instead.** This needs this session to stay open, and the §3
  rules apply unchanged: the same split, the thread replies and base merge the
  user consented to in the F gate, no rebase or force push, and threads are never
  resolved. If the overlay has a `Watch locally` section, run what it names under
  these rules. Otherwise invoke the `loop` skill with `10m` and the §3 prompt
  adapted for a local session — `gh` instead of `mcp__github__*`, work in the
  worktree `$WT` with the explicit PR number, and no auto-fix step (each loop tick
  is a sweep).
- **No watcher.**

## 5 · One-time setup: Slack pings for decisions

The watcher asks for decisions inside its cloud session, and nothing notifies
the user of that by itself. Cloud sessions have no push notifications, and
GitHub does not notify you about replies posted under your own account. So
the watcher posts to a Slack webhook, which `claude-cloud-spawn` sessions read
from their cloud environment:

1. In Slack, create a webhook that lands as a DM to you. The simplest option
   is Workflow Builder → **New workflow** → **From a webhook**, with one text
   variable named `text`, and a step **Send a message to** you containing
   `{text}`. An incoming webhook into a private channel also works if its
   message mentions you.
2. Pin the CLI to one environment. `claude --cloud` does not follow the web
   app's selector: with no `/remote-env` pick it uses the first environment in
   its list, which can be an organization one. Pick yours in `/remote-env`, or
   set `remote.defaultEnvironmentId` in `~/.claude/settings.json`.
3. At claude.ai/code, open that environment. Under **Environment variables**
   add `SHIP_IT_SLACK_WEBHOOK=<the webhook URL>`. Under **Network access →
   Limited → Allowed domains** add `hooks.slack.com`.

Environment variables are readable by anyone who uses that environment, so
keep it a personal one. A running watcher picks the variable up the next time
its VM restores from idle, so no respawn is needed. Test it by asking any
session in that environment to send a ping with the curl command from §3.
