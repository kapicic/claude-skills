# Linear mechanics

Load with `ToolSearch`: `select:mcp__claude_ai_Linear__get_project,mcp__claude_ai_Linear__get_status_updates,mcp__claude_ai_Linear__save_status_update,mcp__claude_ai_Linear__save_issue,mcp__claude_ai_Linear__list_issues`
(the claude.ai Linear connector's names; a locally configured Linear MCP server
exposes the same tools under its own prefix — `ToolSearch "+linear save_issue"`
finds them).

- **Read the last updates:** `get_status_updates` on the project, limit 3.
- `save_status_update` with **no `id`** creates a new update; passing `id` edits that
  one. A pasted `#project-update-<id>` URL usually means "the project this update is
  on", not "edit this update" — confirm which.
- **Health:** `health` is `onTrack` | `atRisk` | `offTrack`.
- `save_issue` requires `team` + `title` on create; `project` accepts the slug from
  the URL. Search first with `list_issues` (its `query` parameter) to avoid a
  duplicate.
- Markdown fields take **literal newlines**, not `\n`. Mentions are `@displayName`.
- Project lookup accepts the full URL slug (`sso-rollout-…-3b2c1b6f77d3`).
- Issue comments: `save_comment` with the issue identifier (`ENG-123`).
