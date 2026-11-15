---
name: check-bots
description: Triage review-bot comments (CodeRabbit, Copilot, Greptile, and any other bot reviewer) on the current branch's PR, recommend fix / skip / discuss for each, wait for approval, apply the approved fixes, then reply to and resolve the threads. Use for "check bots", "check rabbit", "go through CodeRabbit's / Copilot's comments", "address the bot review", or /check-bots.
---

# Triage review-bot comments

You are processing automated review feedback on a PR. Follow these steps strictly and stop at the checkpoint for user approval.

Review bots: `${user_config.review_bots}`. If this is a comma-separated list of GitHub logins, triage only comments from those logins. If it is `auto`, empty, or unsubstituted (the skill runs outside a plugin install), triage every comment whose author GitHub marks as a bot (`user.type == "Bot"`). A bot the user names in the request ("just CodeRabbit") narrows either.

## Step 1: Find the PR for current branch

Run `git branch --show-current` to get the branch name, then capture both PR and repository context — you'll need owner, repo, and PR number for the API calls in later steps:

    gh pr view --json number,url,title,state
    gh repo view --json owner,name --jq '{owner: .owner.login, name: .name}'

Substitute the captured values for `{owner}`, `{repo}`, and `{pull_number}` everywhere they appear in the rest of this command. (`gh api` does NOT auto-resolve these — you must interpolate them yourself.)

If no PR exists for this branch, STOP immediately and tell the user: "No PR found for branch `<branch>`. Create a PR first, then run this command again." Do not continue.

## Step 2: Fetch and assess the bot comments

Get all inline review comments:

    gh api repos/{owner}/{repo}/pulls/{pull_number}/comments --paginate

Keep the comments whose author matches the rule above (`user.login` in the configured list, else `user.type == "Bot"`). Skip replies inside a thread a human already resolved. Also fetch top-level reviews via `gh api repos/{owner}/{repo}/pulls/{pull_number}/reviews` for any summary-level bot feedback, with the same author rule.

If nothing matches, say which bots you looked for and STOP.

For each comment, capture: the bot's login, comment ID, file path, line number, the suggestion itself, and the thread/review ID (needed later for replying and resolving).

### Assess each comment

For every comment, do the following before presenting the table:

1. Read the actual file at the referenced line, plus enough surrounding context to understand what the code is doing (the function, the imports it depends on, how it's called elsewhere if relevant).
2. Evaluate the suggestion against the real code. Consider:
   - Is the bot correct about the issue, or is it a false positive based on missing context?
   - Does the fix actually improve things, or is it stylistic noise?
   - Does it conflict with patterns used elsewhere in the codebase?
   - Is it a real bug, a maintainability win, a nitpick, or wrong?
   - What's the effort vs. value tradeoff?
3. Form a recommendation: **Fix**, **Skip**, or **Discuss** (when it's a judgment call the user should make).

Be opinionated. Review bots generate a lot of noise, the point of this step is to filter it. Don't recommend fixing something just because a bot suggested it. When two bots flag the same line for the same reason, assess it once and list both comment IDs.

### Present the assessment

| # | Bot | File:Line | Bot says | My assessment | Recommendation | Comment ID |
|---|-----|-----------|----------|---------------|----------------|------------|
| 1 | coderabbitai | src/auth.ts:42 | Use const instead of let | Variable is reassigned on line 47, suggestion is wrong | Skip (false positive) | 1234567 |
| 2 | copilot | src/api.ts:88 | Missing null check on user.email | Valid, would throw for OAuth users without email | Fix | 1234568 |
| 3 | coderabbitai | src/utils.ts:15 | Extract to helper | Used once, extraction adds indirection | Skip (nitpick) | 1234569 |
| 4 | greptile | src/db.ts:120 | Potential N+1 query | Real issue but fix requires schema change | Discuss | 1234570 |

After the table, give a one-line summary: e.g. "Recommend fixing 4 of 11 comments. 5 are false positives or nitpicks, 2 need your input on scope."

## Step 3: WAIT for user input

Stop here. Ask the user how to proceed. They can say "go with your recommendations", "fix all", "fix 1, 3, 5", "skip 2", or give specific instructions per comment. Do not proceed without explicit approval.

## Step 4: Apply fixes

For each comment the user approved for fixing:

1. Read the relevant file.
2. Make the fix.
3. Keep a running log entry: comment ID, file:line, what you changed, your reasoning.

For each comment marked Skip (either by your recommendation or user choice), keep a log entry too: comment ID, your reasoning for skipping. You'll still reply to those threads.

If during fixing you discover a suggestion you initially recommended is actually problematic (breaks tests, conflicts with another change, etc.), stop and flag it to the user before continuing.

## Step 5: Commit and push

Stage only the files you modified. Use a commit message like:

    fix: address review-bot feedback

    Addresses comments: #1, #3, #5 on PR #<number>

Then `git push`.

## Step 6: Reply to threads and resolve

For every comment in your log (both fixed and skipped), post a reply in the thread:

    gh api repos/{owner}/{repo}/pulls/{pull_number}/comments/{comment_id}/replies \
      -f body="<your reasoning>"

Reply tone:

- For **fixed** comments: briefly state what you changed and why. Reference the commit SHA. Example: "Fixed in <sha>. Added null check before accessing email since OAuth users may not have one."
- For **skipped** comments: explain the reasoning honestly. Example: "Not applying this. The variable is reassigned on line 47 so const wouldn't compile." Or: "Skipping as nitpick, the function is only used once and extracting it would add indirection without benefit."

Then resolve the threads. Resolving requires GraphQL since REST doesn't support it. First fetch thread node IDs:

    gh api graphql -f query='
      query($owner: String!, $repo: String!, $pr: Int!) {
        repository(owner: $owner, name: $repo) {
          pullRequest(number: $pr) {
            reviewThreads(first: 100) {
              nodes { id isResolved comments(first: 1) { nodes { databaseId } } }
            }
          }
        }
      }' -f owner={owner} -f repo={repo} -F pr={pull_number}

Match `comments.nodes[0].databaseId` to your comment IDs to find the right `thread.id`, then resolve:

    gh api graphql -f query='
      mutation($threadId: ID!) {
        resolveReviewThread(input: {threadId: $threadId}) {
          thread { isResolved }
        }
      }' -f threadId="{thread_node_id}"

**Resolution rules:**

- Resolve threads where you applied the fix.
- Resolve threads you skipped as false positives or nitpicks (your reply explains why).
- Do NOT resolve threads marked "Discuss", leave those open for the user/reviewer.

## Final report

Print a summary:

- Fixed: count and list of comment IDs
- Skipped: count and list with one-line reason each
- Left open for discussion: count and list
- Commit SHA and PR link
- Any threads that failed to reply or resolve (with the error)
