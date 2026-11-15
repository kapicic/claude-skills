# kapicic skills

Claude Code plugins I use every day: a deep stateful PR reviewer, conflict-aware rebasing, review-bot triage, cloud PR sessions, tracker updates, and rewriting AI-sounding text.

## Install

```
/plugin marketplace add kapicic/claude-skills
/plugin install <plugin>@kapicic
```

Then run `/reload-plugins` or restart Claude Code. Update later with `/plugin marketplace update kapicic`.

## Plugins

| Plugin | Skills | What it does |
| --- | --- | --- |
| [`pr-review`](https://github.com/kapicic/pr-review) | `pr-review` | Full or incremental review of a GitHub PR. It remembers earlier rounds, routes the diff to focused review lenses, and posts one consolidated review after you triage the findings. |
| [`unslop`](https://github.com/kapicic/unslop) | `unslop` | Rewrites AI-padded text so it is short and sounds like you, without dropping a fact. |
| [`git-pr`](plugins/git-pr) | `rebase-on-base`, `merge-in-base`, `check-bots` | Rebase onto, or merge in, your PR's base branch, resolving conflicts by intent. Triage review-bot threads (CodeRabbit, Copilot, Greptile, …): recommend, wait for your go, fix, reply and resolve. |
| [`cloud-pr`](plugins/cloud-pr) | `autofix-pr-cloud`, `pr-review-cloud` | Starts a claude.ai cloud session that fixes a named PR's failing CI or reviews it, and hands back the link. Your checkout isn't touched. |
| [`tracker-updates`](plugins/tracker-updates) | `tracker-updates` | Writes project updates, issues and comments in Linear, Jira or GitHub Issues/Projects as 3–5 goal-oriented bullets, and posts only after you approve the draft. |

Skills load as `<plugin>:<skill>`, for example `/git-pr:rebase-on-base`. Asking in plain words ("rebase on latest", "fix CI on PR 123 in the cloud") also triggers them.

## Configuration

Plugins that depend on your tools ask which ones when you enable them. Every option defaults to `auto`, which detects the tool, so you can skip the questions. Change an answer later in `/config`.

| Plugin | Option | Values | `auto` means |
| --- | --- | --- | --- |
| `tracker-updates` | `tracker` | `auto`, `linear`, `jira`, `github` | The pasted link decides, else the connected tracker MCP, else GitHub |
| `git-pr` | `review_bots` | `auto`, or comma-separated logins such as `coderabbitai[bot]` | Every commenter GitHub marks as a bot |
| `pr-review` | `stack_tool` | `auto`, `gh-stack`, `graphite`, `git` | Detected per repo: `gh stack`, then Graphite, then plain git |

## Requirements

- `git-pr` and `cloud-pr` use the [GitHub CLI](https://cli.github.com) (`gh`), logged in.
- `cloud-pr` also needs a `claude` CLI logged in to claude.ai with cloud sessions enabled. The first run in a repo asks you to accept folder trust once. It runs sessions in the first cloud environment in its list, not the one picked in the web app, so pin yours with `/remote-env` or `remote.defaultEnvironmentId` in `~/.claude/settings.json`. `pr-review-cloud` also needs pr-review's claude.ai edition uploaded to claude.ai.
- `tracker-updates` uses the Linear or Atlassian (Jira) connector or MCP server, or `gh` for GitHub Issues/Projects. GitHub project status updates need the `project` token scope: `gh auth refresh -s project`.

Repo-specific details, such as the check command, the formatter and how migrations are named, come from the repo's own `AGENTS.md` or `CLAUDE.md`, not from these skills.

## License

MIT
