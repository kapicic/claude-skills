---
name: autofix-pr-cloud
description: Spawn a CLOUD session that fixes a PR's failing CI and pushes the fix, for a PR you name — the built-in /autofix-pr, but targetable. Use when the user runs /autofix-pr-cloud, or asks to "autofix PR N", "fix CI on <branch>", "autofix that PR in the cloud", naming a PR number, branch name, or GitHub URL. The built-in /autofix-pr only ever acts on the current branch and takes no argument; this is the version that takes one. Hands back a claude.ai link and ends the turn — it never fixes anything locally itself.
---

# /autofix-pr-cloud — autofix any PR, not just the one you're standing on

The built-in `/autofix-pr` reads the current branch, so fixing a different PR
means checking it out first. This skips that: the cloud session fetches the PR
ref itself, so **your working copy is never touched** — no checkout, no stash,
no branch switch. If the user explicitly wants their local checkout moved too,
that is a separate request; do not do it as a side effect.

**Mechanism:** `scripts/claude-cloud-spawn` at the plugin root — two levels up
from this skill's base directory, so `SPAWN="<base dir>/../../scripts/claude-cloud-spawn"`
— starts a plain claude.ai cloud session on the repo you run it from and prints
its link. It wraps `claude --cloud` in a pseudo-terminal (`--cloud` refuses a
non-TTY caller) and launches from a fixed anchor worktree at `origin/<base>`,
because the sandbox seeds from the local checkout's commit and the caller's
branch is usually unpushed. The script's header has the details. Each call is
an independent session, so parallel autofixes don't race.

Run it from inside the PR's repo. It needs `gh`, `git`, `perl`, `script` and a
`claude` CLI logged in to claude.ai with cloud sessions enabled.

## Steps

1. **Resolve the target to a PR number.** Accept any of:
   - a number → use it
   - a GitHub URL → trim to the number after `/pull/`
   - **a branch name** → `gh pr view <branch> --json number,title,headRefName,baseRefName,author,state`
   - nothing → the current branch's PR, same command without the argument

   Confirm the PR is open and get its `headRefName`, `baseRefName` and `title`,
   plus the repo (`gh repo view --json nameWithOwner --jq .nameWithOwner`). No
   PR for that branch → say so and stop; never guess a number.

   If the author is not the user, say whose PR it is and **ask before firing** —
   pushing to someone else's branch is not yours to decide.

2. **Spawn it.** Substitute `<REPO>`, `<N>`, `<BRANCH>` and `<BASE>` into the
   prompt below and pass it through a quoted heredoc so backticks survive:

   ```bash
   prompt=$(cat <<'PROMPT'
   <the autofix prompt below, substituted>
   PROMPT
   )
   "$SPAWN" --base <BASE> "$prompt"
   ```

   It prints one `https://claude.ai/code/session_…` link, or exits non-zero with
   the tail of the CLI output. Takes ~5–10s.

3. **Report one line and END THE TURN:**

   ```
   Spawned cloud autofix on PR #<N> — <title>
     → <link>
   ```

   Then stop. Do not poll, do not wait for the fix, do not start fixing locally.

## The autofix prompt to inject

> Fix the failing CI on PR #`<N>` (branch `<BRANCH>`) in `<REPO>`, then push.
> You start with zero context.
>
> ENVIRONMENT: there is **no `gh` CLI** here — GitHub is only reachable through
> `mcp__github__*` (load with `ToolSearch select:mcp__github__pull_request_read,mcp__github__get_job_logs`).
> Do **not** call `pull_request_read` with `get_files` or `get_diff`: it exceeds
> the tool's token cap on any non-trivial PR. Get the code with git instead:
> `git fetch origin pull/<N>/head:pr-<N> && git checkout pr-<N>`, and diff
> against `$(git merge-base origin/<BASE> pr-<N>)`.
>
> 1. `pull_request_read{method:"get_check_runs"}` on #`<N>`. Nothing with
>    conclusion failure / timed_out / action_required → say so in one line and
>    STOP. Checks still running are not failures; do not wait for them.
> 2. Otherwise pull the failing job's log with `get_job_logs` and diagnose.
>
> YOU MAY fix and push, without asking: lint, formatting, import ordering, and
> a mechanical compile error in a file this PR already touches (a missing
> import, an unused variable). These have one correct answer. Use the lint and
> format commands the repo's `AGENTS.md` / `CLAUDE.md` names, else the ones the
> failing CI job ran.
>
> YOU MUST NOT push anything else — no behaviour or logic changes, no test
> changes, no migration renames or timestamp reordering, no dependency or
> lockfile changes, no config changes, nothing a reviewer asked for. Diagnose
> those and report them instead.
>
> YOU MUST NOT: post any GitHub comment or review; merge, close, reopen or
> change draft status; force-push; or push to any branch other than
> `<BRANCH>`.
>
> BEFORE PUSHING: re-fetch and confirm the head hasn't moved. Push with
> `git push origin pr-<N>:<BRANCH>`. On a non-fast-forward rejection, re-fetch
> and rebase once; if it is rejected again, STOP and report. Never force. If the
> push fails on credentials or permission, report that verbatim.
>
> COMMIT: one commit, `chore(ci): fix lint/format on <BRANCH>`, ending with a
> `Co-Authored-By:` trailer naming the Claude model you are.
>
> REPORT: what was failing, the root cause, whether you pushed (with the sha) or
> why not, and anything you deliberately left for a human. One or two lines is a
> fine report — never invent work to justify a run.

## If it fails

- "no session link" with a folder-trust prompt in the output → the anchor is
  new or lost trust. The error names the anchor path; ask the user to run
  `! cd <anchor path> && claude` once and accept, then retry.
- "requested branch or commit was not found" in the run log → the anchor is not
  on a pushed commit; the script re-detaches it at `origin/<BASE>` each call,
  so check that `git fetch origin <BASE>` succeeded.
- Session runs in the wrong environment (an org one, without your env vars or
  network allowlist) → `claude --cloud` ignores the web app's environment
  picker and, with nothing pinned, uses the first environment in its list. Ask
  the user to pick theirs once with `/remote-env`, or set
  `remote.defaultEnvironmentId` in `~/.claude/settings.json`, then retry.
- Debug a spawned run: `RemoteTrigger` `get_run_log` with the `session_…` id
  from the link (works for plain sessions, not just routine runs).
