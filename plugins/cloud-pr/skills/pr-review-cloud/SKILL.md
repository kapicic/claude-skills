---
name: pr-review-cloud
description: Spawn a CLOUD session that runs the pr-review skill on a PR, then return its claude.ai link immediately — the /autofix-pr pattern, for review. Use when the user runs /pr-review-cloud, or asks to "review this PR in the cloud", "kick off a cloud review", "review PR N remotely / in the background / on the server". NOT for a normal review — plain "/pr-review N" or "review PR N" means the local skill, which is deeper. This one hands back a link and ends the turn; it never reviews anything itself.
---

# /pr-review-cloud — fire a cloud review and hand back the link

**Mechanism:** `scripts/claude-cloud-spawn` at the plugin root — two levels up
from this skill's base directory, so `SPAWN="<base dir>/../../scripts/claude-cloud-spawn"`
— starts a plain claude.ai cloud session on the repo you run it from and prints
its link. It wraps `claude --cloud` in a pseudo-terminal (`--cloud` refuses a
non-TTY caller) and launches from a fixed anchor worktree at `origin/<base>`,
because the sandbox seeds from the local checkout's commit and the caller's
branch is usually unpushed. The script's header has the details.

**Needs the pr-review skill on claude.ai.** Cloud sandboxes do not get your
local skills or plugins — they sync claude.ai Skills. Upload the claude.ai
edition from https://github.com/kapicic/pr-review (Customize → Skills) first.

## Steps

1. **Resolve the target.** A number (or several, for a stack) from the user's
   arguments; a GitHub URL trimmed to its number. With no argument, take the
   current branch's PR:

   ```bash
   gh pr view --json number,title,url,baseRefName --jq '"\(.number) \(.baseRefName) \(.title)"'
   ```

   No argument and no PR on this branch → say so and stop. Do not guess, and do
   not review the branch diff instead. For a named PR, get its base with
   `gh pr view <N> --json baseRefName --jq .baseRefName`.

2. **Spawn it.** Pass the user's `model=` / `effort=` through untouched — the
   skill parses them itself:

   ```bash
   prompt=$(cat <<'PROMPT'
   /pr-review <PR…> <any model=/effort= the user passed>

   (cloud-session notes: this session can reach only this repo, so a separate
   review-state repo is out of scope — skip the state read and write, run a
   FULL review, and don't ask about it. Don't call pull_request_read method
   get_files/get_diff for the file list or diff — it exceeds the tool's token
   cap on any non-trivial PR. Fetch the PR ref directly
   (git fetch origin pull/<N>/head, diff against merge-base) instead.)
   PROMPT
   )
   "$SPAWN" --base <BASE> "$prompt"
   ```

   It prints one `https://claude.ai/code/session_…` link, or exits non-zero with
   the tail of the CLI output. Takes ~5–10s.

3. **Report one line and END THE TURN:**

   ```
   Spawned cloud pr-review on PR #<N> — <title>
     → <link>
   ```

   Then stop. Do not poll, do not wait for findings, do not start a local
   review as well.

## What the cloud session actually runs

`/pr-review` there resolves to the **claude.ai edition**: one context walking
the lenses, rather than parallel subagents plus a completeness critic. It says
as much in its own first message.

It is also **stateless**: a plain session gets exactly one repo, and
`--add-dir` does not add a second source, so a review-state repo is out of
scope. Every cloud round is a FULL review with no prior rounds and no review
preferences, and nothing is recorded. The local `/pr-review` is the stateful,
deeper pass.

Tell the user this only if they seem to expect local depth or an incremental
round — otherwise the link is the answer.

## If it fails

- "no session link" with a folder-trust prompt in the output → the anchor is
  new or lost trust. The error names the anchor path; ask the user to run
  `! cd <anchor path> && claude` once and accept, then retry.
- "requested branch or commit was not found" in the run log → the anchor is not
  on a pushed commit; the script re-detaches it at `origin/<BASE>` each call,
  so check that `git fetch origin <BASE>` succeeded.
- Debug a spawned run: `RemoteTrigger` `get_run_log` with the `session_…` id
  from the link (works for plain sessions, not just routine runs).
