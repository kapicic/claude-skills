---
name: rebase-on-base
description: Rebase the current branch onto the latest version of its base branch — the one named, else the open PR's base, else the remote's default branch — resolving conflicts by the intent of both sides. Use for "rebase", "rebase onto main/staging/develop", "rebase on latest", or /rebase-on-base.
---

# Rebase onto the base branch

You are rebasing the current branch onto the latest `origin/<base>`. Resolve
conflicts by understanding the intent of both sides — never blindly take one side.

## Step 1: Pre-flight checks

Run in parallel:

    git branch --show-current
    git status --porcelain
    git log --oneline -5

- If `git status --porcelain` shows any output, STOP. The working tree must be clean. Tell the user: "Working tree is dirty. Commit or stash changes before rebasing." and list the dirty paths.
- Capture the current branch name as `<branch>` and HEAD SHA as `<original-sha>` — you may need them for recovery.
- Ensure `git rerere` (reuse recorded resolution) is enabled — same conflicts often recur across the commits being replayed, and rerere auto-applies prior resolutions:

      git config rerere.enabled

  If the value is empty or `false`, enable it for this repo:

      git config rerere.enabled true

  Do **not** enable `rerere.autoupdate` — keeping it off means `git add` is still required after a recorded resolution is applied, which forces a verification read. (To enable rerere once for all repos, the user can run `git config --global rerere.enabled true` themselves.)

## Step 2: Resolve the base branch

Take the first that applies:

1. The branch the user named (`/rebase-on-base develop`).
2. The base of this branch's open PR:

       gh pr view --json baseRefName --jq .baseRefName

3. The remote's default branch:

       git symbolic-ref --short refs/remotes/origin/HEAD | sed 's|^origin/||'

Call it `<base>` and state in one line where it came from. If `<branch>` is
`<base>` or the remote's default branch, STOP and tell the user — never rebase a
trunk branch.

## Step 3: Fetch the latest base

    git fetch origin <base>

Then check whether the branch is already up to date:

    git log --oneline <branch>..origin/<base> | head -5
    git log --oneline origin/<base>..<branch> | head -5

If `origin/<base>` has no commits beyond the merge base, tell the user "Already up to date with origin/<base>" and stop.

## Step 4: Start the rebase

Read the repo's `AGENTS.md` or `CLAUDE.md` once before starting: it usually says
how this repo regenerates lockfiles and generated code, names migrations, runs
its formatter, and which check command covers a change. Use those over the
generic defaults below.

    git rebase origin/<base>

If it succeeds without conflicts, skip to Step 6.

## Step 5: Resolve conflicts (loop until clean)

For each conflict that `git rebase` reports:

1. Run `git status` to see the conflicting files. Then check whether `rerere` has pre-resolved any of them:

       git rerere status
       git rerere diff

   Any path listed by `rerere status` had a previously-recorded resolution auto-applied (the conflict markers may already be gone). **Verify** each one by reading the file — a stale recording will silently re-introduce a bug. If a recorded resolution is wrong, run `git rerere forget <path>` and re-resolve manually.
2. For each conflicting file:
   - Read the file to see the `<<<<<<<` / `=======` / `>>>>>>>` markers.
   - Use `git log --oneline <original-sha> -- <file>` and `git log --oneline origin/<base> -- <file>` to understand recent intent on each side.
   - When the conflict is non-trivial, also read the surrounding code (the function, the imports, related callers) so your resolution preserves behavior from both sides.
   - **Resolution principle:** preserve the intent of BOTH sides where possible. Don't default to "ours" or "theirs" — combine them. Drop one side only when the change is genuinely superseded.
   - Pay particular attention to:
     - **`ours` / `theirs` are swapped during rebase.** Git replays your commits on top of `origin/<base>`, so during a rebase `--ours` refers to `origin/<base>` (the new base) and `--theirs` refers to your local commits being replayed — the opposite of a merge. Keep this in mind for every `git checkout --ours` / `--theirs` decision below.
     - **Lockfiles** (`pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `Cargo.lock`, `uv.lock`, `poetry.lock`): never hand-edit. Take the base's lockfile — during rebase that's `git checkout --ours <lockfile>` — then regenerate it with the package manager's install command so it reflects the merged manifest. If the manifest itself (`package.json`, `Cargo.toml`, `pyproject.toml`) conflicts, resolve **that** first: don't use `--ours` / `--theirs` on it; merge the dependency lists from both sides so no entry is lost, then regenerate the lockfile.
     - **Migrations**: if both sides added migrations with clashing timestamps, sequence numbers or names, keep both files and renumber the local one to sort after the base's, following the repo's naming scheme, so ordering is deterministic.
     - **Generated files** (generated types or clients, `dist/`, snapshots): regenerate rather than merge, using the repo's generate or build command, after resolving the source files.
     - **Auto-formatted files**: re-run the repo's formatter on the file after manual resolution to normalize whitespace.
   - Edit the file to remove all conflict markers and produce the merged content.
3. Stage the resolved file: `git add <file>`.
4. After all files in this conflict step are resolved, continue: `git rebase --continue`.
5. If more conflicts appear, repeat from step 1 of this loop.

If at any point you cannot confidently resolve a conflict (e.g., both sides made fundamentally incompatible changes to the same logic), STOP. Do NOT guess. Run `git rebase --abort` to restore the original state, then tell the user exactly which file and which hunk needs their judgment, and quote both sides.

If the rebase becomes more tangled than expected (e.g. >5 conflicting commits, or the same file conflicts repeatedly across multiple commits), stop and ask the user whether to continue, or abort and merge the base in instead (the `merge-in-base` skill).

## Step 6: Verify the result

After the rebase finishes cleanly, run in parallel:

    git status
    git log --oneline -10
    git diff origin/<base>...HEAD --stat

Sanity-check that:

- `git status` shows a clean working tree on `<branch>`.
- The commits ahead of `origin/<base>` match what was on the branch before (same count or fewer if commits were squashed/dropped).
- The diff stat looks reasonable — no surprise massive deletions.

If anything looks wrong, tell the user and offer to reset: `git reset --hard <original-sha>`.

## Step 7: Run checks on touched files

If any conflicts were resolved, run the cheapest check that covers the resolved
files — the one the repo's `AGENTS.md` / `CLAUDE.md` names, otherwise its
`check`, `lint` or `typecheck` script (or the language's equivalent). This
catches type regressions or formatting issues introduced by the resolution. If
the checks fail, fix the issues (or revert if the fix isn't obvious) before
declaring done.

## Final report

Print a summary:

- Branch: `<branch>` onto `origin/<base>` (and where `<base>` came from)
- Original HEAD: `<original-sha>` → New HEAD: `<new-sha>`
- Commits rebased: count
- Conflicting files resolved: list with one-line note per file on the resolution approach
- Check result: the command run, pass/fail
- Reminder: if the branch was already pushed, the user will need `git push --force-with-lease` to update the remote. Do NOT push automatically.
