---
name: tracker-updates
description: This skill should be used when writing project status updates, issue descriptions, or issue comments in an issue tracker — Linear, Jira, or GitHub Issues/Projects. It enforces a laconic 3-5 bullet, goal-oriented house style, a draft-then-confirm flow, and each tracker's posting mechanics (which tool creates vs edits, health values, required fields). Triggers on "post a project update", "update this project", "add an issue for that", "update Linear", "update Jira", or a pasted linear.app, atlassian.net/browse, or github.com issue/project URL.
---

# Tracker Updates

## Purpose

Produce tracker updates that a lead can read in fifteen seconds and know: what moved
toward the goal, what is in the way, and whether the date still holds. Optimize for
the reader who did not attend the standup.

## Pick the tracker

Tracker: `${user_config.tracker}`. If this is `auto`, or not one of `linear`, `jira`,
`github` (it stays unsubstituted when the skill runs outside a plugin install),
decide in this order:

1. **A pasted URL wins:** `linear.app` → linear; `*.atlassian.net/browse/…` or
   `/jira/…` → jira; `github.com/<owner>/<repo>/issues/…` or
   `github.com/(orgs|users)/<owner>/projects/…` → github.
2. **The configured value**, when it is one of the three.
3. **The tracker MCP that is connected and signed in:** Linear tools
   (`ToolSearch "+linear save_issue"`), or Atlassian tools beyond
   `authenticate` (`ToolSearch "+atlassian jira"`). If both are, ask once which one
   this project lives in.
4. **GitHub** (`gh`), when no tracker MCP is connected.

Then read `references/<tracker>.md` for the posting mechanics. Everything else in
this file applies to every tracker.

## The format

**3-5 bullets. No headers, no preamble, no closing summary.** A sixth bullet means
two of them were activity, not outcome — cut them.

Each bullet carries one of these, and nothing else:

- **A goal moved** — what now works that did not before, with the artifact linked
  (PR, release, issue). Not "worked on X".
- **A decision** — what was decided, and the reason in one clause. Reason first if
  it is the surprising part.
- **A blocker** — what is blocked, on whom or what, and what happens next.
- **A health call** — why the date or health changed, stated plainly.

Order by what the reader acts on, not chronologically.

## Writing rules

- **Goal, not activity.** "Single sign-on is in: SAML login (#812) and SCIM
  provisioning (#840) merged" — not "continued work on SSO".
- **Every claim carries its evidence.** Link the PR, release, or issue inline.
  A bullet with no artifact and no decision is filler.
- **Lead the bullet with the noun that matters**, bolded when it is a topic shift
  (`**Renaming Boards → Spaces.**`). Skip bold on continuation bullets.
- **Name names and numbers.** "Acme Corp is blocked on SCIM provisioning" beats
  "a customer is blocked". Dates, counts, versions, flag keys — concrete over vague.
- **State bad news first and flatly.** "Still at risk: the Mar 14 target passed
  while the team was pulled onto the billing migration." No cushioning, no
  "unfortunately", no recovery-plan padding nobody asked for.
- **No restating the project description.** The reader has it. Write only the delta
  since the last update.
- **Cut every adjective that is not load-bearing.** No "significant", "great
  progress", "exciting", "successfully".

## Workflow

1. **Read the last update first** (the last 3, with the tracker's mechanics). The new
   update is a delta on it — never repeat a bullet that already shipped, and carry
   forward an unresolved blocker only if its state changed.
2. **Verify against ground truth, not conversation memory.** Check the PR state
   (`gh pr view <n> --json state,reviewDecision,title`), the merge, the release, the
   code. A PR described as "in review" that is actually CHANGES_REQUESTED or already
   merged makes the whole update untrustworthy.
3. **When the update references a code change, scope it in the repo** before
   claiming size or surface area. Grep the actual flag keys, tool names, table
   columns, and directories — naming them specifically is what makes an update
   useful instead of decorative.
4. **Draft in chat and wait.** Post nothing until the human approves the text.
   Surface any decision the draft had to assume as an explicit question with a
   recommendation — depth of a rename, whether health changes, which team owns it.
5. **Post, then report back the URLs.**
6. **Flag what is now stale.** A rename or pivot usually leaves the project name,
   description, or milestone titles contradicting the update just posted. Say so and
   offer to fix it; do not fix it unasked.

## Health

Every tracker gets the same three-way call: **on track**, **at risk**, **off track**.
Carry the previous value forward unless something actually changed, and say in a
bullet why it changed. The tracker reference maps it to the tracker's own field, or
says where to state it when the tracker has none.

## Issues

Same laconicism, different shape.

- **Title**: imperative outcome, no ticket-speak. "Rename Boards → Spaces
  across product, code, and docs".
- **Body**: why (one paragraph, lead with the forcing reason) → **Scope** (bulleted
  by surface, each with the concrete file/key/table it touches) → **Out of scope**
  (name the lookalike that must NOT change — this is the bullet that saves a day).
- **Call out the non-obvious risk inline** in the scope bullet it belongs to:
  orphaned feature-flag override rows, historical rows keeping an old enum value,
  broken analytics charts, a migration needing its own PR.
- **Size**: state it in prose ("not a small change: spans X, Y, Z"). Do not set an
  estimate unless that team demonstrably uses estimates — check a few existing
  issues first; many teams leave them empty.
- Set priority and project (or epic, or milestone) when the tracker has them; leave
  assignee and status alone unless told.
- Before creating, search for an existing issue on the same thing.

## Worked example

```markdown
- **Renaming Boards → Spaces.** "Board" collides with a term sales already uses
  for something else, so the product concept becomes **Spaces**. Tracked in
  [ENG-1042](…) — not a small change: user-facing copy, the `boards` feature flag,
  the `board` API resource, the `items.kind = 'board'` value (migration +
  backfill), public docs, and analytics events.
- SCIM provisioning fix ([PR #912](…)) is in review — unblocks Acme Corp, still the
  top priority to land.
- Still at risk: the Mar 14 target date has passed while the team was pulled onto
  higher-priority billing work and two prod incidents.
```

Three bullets: a decision with its scope, a blocker with its stake, a health call
with its cause. Nothing else earned a line.
