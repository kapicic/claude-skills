---
name: linear-updates
description: This skill should be used when writing Linear project status updates, issue descriptions, or issue comments — anything posted into Linear. It enforces a laconic 3-5 bullet, goal-oriented house style, a draft-then-confirm flow, and the Linear MCP mechanics (which tool creates vs edits, health values, required fields). Triggers on "update this Linear project", "post a project update", "add an issue for that", a pasted linear.app URL, or "update Linear".
---

# Linear Updates

## Purpose

Produce Linear updates that a lead can read in fifteen seconds and know: what moved
toward the goal, what is in the way, and whether the date still holds. Optimize for
the reader who did not attend the standup.

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

1. **Read the last update first** (`get_status_updates`, limit 3). The new update is
   a delta on it — never repeat a bullet that already shipped, and carry forward an
   unresolved blocker only if its state changed.
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
- Set `priority` and `project`; leave assignee and status alone unless told.

## Linear MCP mechanics

Load with `ToolSearch`: `select:mcp__claude_ai_Linear__get_project,mcp__claude_ai_Linear__get_status_updates,mcp__claude_ai_Linear__save_status_update,mcp__claude_ai_Linear__save_issue,mcp__claude_ai_Linear__list_issues`
(the claude.ai Linear connector's names; a locally configured Linear MCP server
exposes the same tools under its own prefix).

- `save_status_update` with **no `id`** creates a new update; passing `id` edits that
  one. A pasted `#project-update-<id>` URL usually means "the project this update is
  on", not "edit this update" — confirm which.
- `health`: `onTrack` | `atRisk` | `offTrack`. Carry the previous value forward
  unless something actually changed, and say in a bullet why it changed.
- `save_issue` requires `team` + `title` on create; `project` accepts the slug from
  the URL.
- Markdown fields take **literal newlines**, not `\n`. Mentions are `@displayName`.
- Project lookup accepts the full URL slug (`sso-rollout-…-3b2c1b6f77d3`).

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
