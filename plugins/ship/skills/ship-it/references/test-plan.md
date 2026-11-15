# Test plan — where and how, asked per feature

`ship` phase 6 runs the repo's lint, typecheck and test suites. For some changes
that is enough. For others it proves nothing: a migration that only breaks on a
real database, a toolbar that only overflows in a real browser, a classifier that
only misfires against a real vendor, a daemon that only fails on its target OS.
Which of those a feature needs depends on the feature, and what can actually be
reached (a preview, a test account, a vendor sandbox, a real connection) is
something only the human knows. So the plan asks.

## 1 · Build the candidates from the MAP

Inputs: the plan's file list, the phase-2 blast-radius table (its interleavings
and siblings), and the one-sentence behavior. Match the touched surfaces to rows
below, plus the rows in the overlay's `Test plan` section; each matched row
contributes WHERE and HOW candidates. Candidates are drafts — rewrite each HOW in
this feature's own terms before asking.

| Surface in the diff | WHERE candidates | HOW candidates (make them concrete) |
|---|---|---|
| Pure logic, helper, classifier | unit — a test file beside the source | one case per sibling shape from the MAP; the boundary the helper exists to guard |
| Repository, raw SQL, tenant or owner scope | integration test against a real database (testcontainers, or the repo's harness) | seed rows for two tenants; assert the other tenant's row never comes back |
| Migration | local DB (the repo's migrate check; migrate up locally); a PR preview with its own database, if the repo has one | apply it, then exercise the route that depends on it |
| Backend route or service | in-process HTTP test (the framework's `inject`, supertest); local stack; PR preview | `curl` the route with real auth; force the failure path (expired token, 404, 429) |
| Public API, SDK or CLI | the route rows above, plus the client against the preview | the client command end to end; the API diff check, if the repo runs one |
| Frontend component or flow | local app in a real browser (`claude-in-chrome`, Playwright); PR preview | walk happy / empty / error / loading states with a screenshot of each; measure geometry for layout work |
| Third-party integration | a captured real payload in a unit test; a real account or vendor sandbox | call it with a real account; check the error path |
| Background job, queue, cron | the job's test harness; a local worker; a preview or staging environment | enqueue one job, then read its logs and the rows it wrote |
| Infrastructure, CI workflow, Terraform | `terraform plan`; `workflow_dispatch` on the branch; script tests | quote the plan; dry-run the job |
| Feature flag | an environment where the flag can be flipped for one test account | test with the flag on and off |
| New log line or metric | an environment that ships telemetry (preview, staging) | query the new line or metric and check its labels |
| User journey across pages | the repo's e2e suite, against the preview | name the spec, and say whether it is new or existing |

Every **interleaving** in the MAP adds one HOW candidate: a test that forces that
ordering (a deferred promise, a paused mock), named with its file.

A docs-only or copy-only change skips the gate. Say so in one line, with the
artifact check that replaces it.

## 2 · Ask — batched into the phase-3 plan gate

Put these in the same `AskUserQuestion` call as the plan's decisions question.
Each question has at most 4 options; when more rows match, keep the 4 that best
exercise the behavior and page the rest.

- **WHERE — `multiSelect: true`.** One option per environment that applies. The
  recommended set goes first, marked `(Recommended)`: the cheapest set that
  exercises the behavior end to end. Each description says what that
  environment proves that a cheaper one cannot, and what it costs (for example,
  "PR preview — needs the push plus the deploy").
- **HOW — `multiSelect: true`.** One option per concrete scenario, named for this
  feature. For example: "Force the finish-before-release race in
  `complete-job.test.ts`", "Walk Settings → Reconnect on the preview and
  screenshot each state", "`curl` `POST /v1/jobs` with an expired API key".
  Never a generic option like "add tests" or "manual QA".
- **WITH WHAT — only when a chosen environment needs it.** Which test account,
  workspace, connection or vendor sandbox to use. Build the options from what you
  know; the automatic "Other" option lets the human name one.

Ask nothing the code already settles. For example, if the package has no
integration harness, do not offer one; say so instead. Each answer becomes a
`T` line in the plan, beside the `D` lines, and changes the same way (one
follow-up question per line picked):

```
T1  unit — gmail/graph/one-platform 403 shapes in classify-403.test.ts
T2  preview — reconnect a Gmail account with a narrowed scope; screenshot the banner   [post-push]
```

## 3 · Split by when each test can run

- **Pre-push** — unit, integration, local DB, local app, and package suites.
  These run in `ship` phase 6, with their output quoted.
- **Post-push** (tag the T line `[post-push]`) — anything that needs the draft
  PR's deploy or a real third-party account. These run in stage E, after triage,
  so they exercise the code that will actually ship. Get the preview URL from the
  PR (its deploy comment or deployment status), and wait for the deploy with
  `Monitor`, not a sleep loop.

## 4 · Run, then record the evidence

Each T line ends in one of three states:

- **ran** — say what was run, what happened, and point to the evidence: quoted
  output, a screenshot or GIF path, a job or run id, or a log query.
- **failed** — treat it as a finding. Fix it and re-run the T lines the fix
  invalidates.
- **could not run** — for example, the preview deploy is red, access is
  missing, or the vendor sandbox is down. Ask with `AskUserQuestion`: hold,
  swap in another environment, or waive with a reason. Never drop it silently.

The results go in the PR body under **Tested**, one line per T. A reviewer who
can see how a change was tested does not ask how it was tested.

**Guardrails:** say what test data you are about to write, and where. A preview
that shares a database with another environment (the overlay says whether it
does) gets the same care as that environment. Never use a customer's account or
data, and never test against production. Nothing from this stage is posted to
Slack or GitHub except the PR body itself.
