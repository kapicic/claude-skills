---
name: merge-in-base
description: Merge the latest version of the current branch's base branch INTO it — the base named, else the open PR's base, else the remote's default branch — resolving conflicts by the intent of both sides. Use for "merge in staging/main", "merge the base in", "bring my branch up to date without rebasing", or /merge-in-base.
---

# Merge the base branch in

You are merging the latest `origin/<base>` INTO the current feature branch (not
the other way around). Resolve conflicts by understanding the intent of both
sides — never blindly take one side.

## Step 1: Pre-flight checks

Run in parallel:

    git branch --show-current
    git status --porcelain
    git log --oneline -5

- If `git status --porcelain` shows any output, STOP. The working tree must be clean. Tell the user: "Working tree is dirty. Commit or stash changes before merging." and list the dirty paths.
- Capture the current branch name as `<branch>` and HEAD SHA as `<original-sha>` — you may need them for recovery.
- Ensure `git rerere` (reuse recorded resolution) is enabled — if a prior merge or rebase recorded the same conflict, rerere auto-applies the resolution:

      git config rerere.enabled

  If the value is empty or `false`, enable it for this repo:

      git config rerere.enabled true

  Do **not** enable `rerere.autoupdate` — keeping it off means `git add` is still required after a recorded resolution is applied, which forces a verification read. (To enable rerere once for all repos, the user can run `git config --global rerere.enabled true` themselves.)

## Step 2: Resolve the base branch

Take the first that applies:

1. The branch the user named (`/merge-in-base develop`).
2. The base of this branch's open PR:

       gh pr view --json baseRefName --jq .baseRefName

3. The remote's default branch:

       git symbolic-ref --short refs/remotes/origin/HEAD | sed 's|^origin/||'

Call it `<base>` and state in one line where it came from. If `<branch>` is
`<base>` or the remote's default branch, STOP and tell the user — this brings the
base INTO a feature branch, not the reverse.

## Step 3: Fetch the latest base

    git fetch origin <base>

Then check whether the branch is already up to date:

    git log --oneline <branch>..origin/<base> | head -5

If there are no commits in `origin/<base>` beyond the current branch's merge base, tell the user "Already up to date with origin/<base>" and stop.

## Step 4: Start the merge

Read the repo's `AGENTS.md` or `CLAUDE.md` once before starting: it usually says
how this repo regenerates lockfiles and generated code, names migrations, runs
its formatter, and which check command covers a change. Use those over the
generic defaults below.

    git merge --no-ff origin/<base>

Use `--no-ff` so the integration is recorded as a real merge commit, even when fast-forward is possible. This makes the history clearer for reviewers and easier to revert if the merge introduces problems.

If it succeeds without conflicts, skip to Step 7.

## Step 5: Resolve conflicts (single pass)

Unlike rebase, a merge produces conflicts in a single batch. Run:

    git status
    git rerere status
    git rerere diff

Any path listed by `rerere status` had a previously-recorded resolution auto-applied (the conflict markers may already be gone). **Verify** each one by reading the file — a stale recording will silently re-introduce a bug. If a recorded resolution is wrong, run `git rerere forget <path>` and re-resolve manually.

For each conflicting file:

1. Read the file to see the `<<<<<<<` / `=======` / `>>>>>>>` markers. By convention in a merge:
   - `<<<<<<< HEAD` to `=======` is **your branch's** version (`<branch>`).
   - `=======` to `>>>>>>> origin/<base>` is the **incoming** version from the base.
2. Use `git log` on each side to understand recent intent:
   - `git log --oneline -10 HEAD -- <file>`
   - `git log --oneline -10 origin/<base> -- <file>`
3. When the conflict is non-trivial, also read the surrounding code (the function, the imports, related callers) so your resolution preserves behavior from both sides.
4. **Resolution principle:** preserve the intent of BOTH sides where possible. Don't default to "ours" or "theirs" — combine them. Drop one side only when the change is genuinely superseded.
5. Pay particular attention to:
   - **Lockfiles** (`pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `Cargo.lock`, `uv.lock`, `poetry.lock`): never hand-edit. Take the incoming version (`git checkout --theirs <lockfile>`), then regenerate it with the package manager's install command so it reflects the merged manifest. If the manifest itself (`package.json`, `Cargo.toml`, `pyproject.toml`) conflicts, resolve that first by merging both dependency lists.
   - **Migrations**: if both sides added migrations with clashing timestamps, sequence numbers or names, keep both files and renumber your local one to sort after the base's, following the repo's naming scheme, so ordering is deterministic.
   - **Generated files** (generated types or clients, `dist/`, snapshots): regenerate rather than merge, using the repo's generate or build command, after resolving the source files.
   - **Auto-formatted files**: re-run the repo's formatter on the file after manual resolution to normalize whitespace.
6. Edit the file to remove all conflict markers and produce the merged content.
7. Stage the resolved file: `git add <file>`.

If at any point you cannot confidently resolve a conflict (e.g., both sides made fundamentally incompatible changes to the same logic), STOP. Do NOT guess. Run `git merge --abort` to restore the original state, then tell the user exactly which file and which hunk needs their judgment, and quote both sides.

## Step 6: Complete the merge

Once all conflicts are resolved and staged:

    git status

Verify no `Unmerged paths` remain. Then commit with a clear merge message:

    git commit -m "merge: bring origin/<base> into <branch>"

## Step 7: Verify the result

Run in parallel:

    git status
    git log --oneline -10
    git diff <original-sha>...HEAD --stat

Sanity-check that:

- `git status` shows a clean working tree on `<branch>`.
- The merge commit is present and parents the prior HEAD plus `origin/<base>`.
- The diff stat looks reasonable — no surprise massive deletions.

If anything looks wrong, tell the user and offer to reset: `git reset --hard <original-sha>`.

## Step 8: Run checks on touched files

If any conflicts were resolved, run the cheapest check that covers the resolved
files — the one the repo's `AGENTS.md` / `CLAUDE.md` names, otherwise its
`check`, `lint` or `typecheck` script (or the language's equivalent). This
catches type regressions or formatting issues introduced by the resolution. If
the checks fail, fix the issues (or revert if the fix isn't obvious) before
declaring done.

## Final report

Print a summary:

- Branch: `<branch>`, merged `origin/<base>` (and where `<base>` came from)
- Original HEAD: `<original-sha>` → New HEAD: `<new-sha>` (merge commit)
- Commits brought in from the base: count
- Conflicting files resolved: list with one-line note per file on the resolution approach
- Check result: the command run, pass/fail
- Reminder: a regular `git push` will work (no force-push needed since this is a merge commit, not a rewrite).
