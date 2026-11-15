# Stage C — dispatching the fresh review

The reviewer must not share this conversation's context. That is the whole point
of the stage: an agent that watched the code get written reviews the *intent*, and
the intent is exactly what is not in question. So everything below runs in a
worktree, in a separate agent, and returns findings — not narrative.

## 0 · Find pr-review

Stage C drives the `pr-review` plugin's scripts directly. Find them — a user-level
install first, then the newest plugin-cache copy:

```bash
SK=~/.claude/skills/pr-review/scripts
[ -x "$SK/state.py" ] || SK="$(ls -td ~/.claude/plugins/cache/*/pr-review/*/skills/pr-review/scripts 2>/dev/null | head -1)"
[ -x "$SK/state.py" ] || echo "pr-review not installed"
PRR="$SK/.."   # pr-review's skill directory: SKILL.md and references/
```

Not installed → stop and tell the user: `/plugin install pr-review@kapicic`.
`$SK` and `$PRR` below are these paths, resolved — write them out literally in any
prompt you hand to an agent.

## 1 · Size the review — pr-review's `profile`

Run after the draft PR exists and its branch is pushed. The parent builds the
context and worktree with pr-review's own scripts. This is cheap: only a small
JSON result enters context.

```bash
CTX="$("$SK/pr-context.sh" resolve "$PR" | jq -r .ctxFile)"
"$SK/pr-context.sh" worktree --ctx "$CTX" >/dev/null
PROFILE="$("$SK/pr-context.sh" profile --ctx "$CTX" --set net)"
```

Resolve `$EFFORT` exactly as `$PRR/SKILL.md` Phase 2 ("Resolve `$EFFORT` and the
shards") says. Start from `$PROFILE.tier`. Raise it by at most one step, only with
a named reason, and never past `xhigh`. You wrote this code, so you know where it
is fragile better than the line counts do. A Blocker or High that the phase-5 hunt
found and fixed is a named reason on its own. The size and risk rules, including
the high-risk paths, live in `profile` and pr-review's config; do not keep a
second copy here.

Announce the result: `self-review: xhigh (auto — 1,062 code lines; risk:
migration, auth)`. If the human wants a different level, including `max`, that is
a one-word override; do not re-derive it.

| `$EFFORT` | Dispatch |
|---|---|
| `low` / `medium` / `high` | §2 — one fresh subagent runs `pr-review` |
| `xhigh`, or an explicit `max` | §3 — `Workflow` from the parent, reusing this worktree and `$PROFILE.shards` |

## 2 · Dispatch — `low` / `medium` / `high`: one fresh subagent

First remove the parent's worktree (`"$SK/pr-context.sh" cleanup --ctx "$CTX"`).
The subagent's `pr-review` run builds its own worktree at the same path.

`Agent`, `subagent_type: "general-purpose"`. The subagent runs the real
`pr-review` skill, so the lens list, checklist, comment style and scoring rubric
stay single-sourced there. Pass the **resolved** tier, not `auto`. Given `auto`,
the subagent would lose any step you raised it by, and the reason for it.

> Invoke the `pr-review` skill for PR #`<N>` in `<OWNER>/<REPO>` with
> `effort=<resolved tier> mode=full`. Record `"effortReason": "<the reason
> from the announcement>"` in the round. Run Phases 0, 2, 4, 5 and 6 exactly as written.
> (Phase 1 is the interactive picker — not used here. Phase 3 is incremental
> mode's fix verification — `mode=full` skips it.)
>
> Two deviations, both mandatory:
> - **Skip Phase 7 steps 1-4.** Post nothing to GitHub — no review, no comment,
>   no verdict. This is a pre-review of the author's own draft.
> - **Still do Phase 7 step 5** — persist the round with
>   `scripts/state.py append`, with `"kind": "self"` and `"posted": false` —
>   and step 7, the worktree cleanup.
>
> You did not write this code and have no context on why it exists beyond the PR
> body and the issue it links. Do not go looking for that context —
> reviewing it cold is the job.
>
> Then remove the worktree (Phase 7 step 7) and **return only**: the state file
> path, a count per severity, and the one-line claim of each Blocker/High. No
> diff, no narrative, no reproduced findings — the parent reads the state file.

## 3 · Dispatch — `xhigh` (or explicit `max`): `Workflow`

At these levels the point is the per-call reasoning-effort dial, which only
`Workflow`'s `agent()` exposes. **This skill's instructions are the opt-in** the
Workflow tool requires — you do not need to ask again, but do say which level you
are running and roughly how many agents it will spawn.

Reuse §1's worktree:

```bash
WT="$(jq -r .worktree "$CTX")"; RANGE="$(jq -r .range "$CTX")"
"$SK/pr-context.sh" files --ctx "$CTX" --set net     # the changed-file list
jq -c '.shards | map({areas, files})' <<<"$PROFILE"  # [] below 1,500 code lines
```

Classify lenses per `$PRR/SKILL.md` Phase 4 step 2: every Always lens, plus every
routed lens whose surface the diff touches (`$PROFILE.surfaces` is a starting
point), and when unsure, run it. When `shards` is non-empty, A+B gets one entry
per shard, each told to report only on its shard's files. Then:

```js
export const meta = {
  name: 'ship-it-self-review',
  description: 'Fresh-context review of a draft PR: routed lenses, adversarial verification, completeness critic',
  phases: [{ title: 'Find' }, { title: 'Verify' }, { title: 'Critic' }],
}

const WT = '<worktree path>'
const RANGE = '<BASE>..HEAD'
const CHANGED = `<newline-separated changed-file list>`
const LENSES_DOC = '<$PRR resolved>/references/review-lenses.md'
const EFFORT = 'xhigh'   // 'max' only on an explicit human override

const CTX = `Worktree: ${WT}. Review range: ${RANGE}.
Changed files (report ONLY on these; read anything for context):
${CHANGED}
You did not write this code. Review it cold.`

// One agent per lens GROUP, exactly as pr-review Phase 4 groups them.
const LENSES = [
  // A+B: one entry per shard when $PROFILE.shards is non-empty, e.g.
  //   { key: 'ab-1', doc: 'Lens A+B — Correctness & subtle bugs', only: [...shard files] }
  { key: 'ab', doc: 'Lens A+B — Correctness & subtle bugs' },
  { key: 'd',  doc: 'Lens D — Test adequacy (mutation mindset)' },
  // …plus each routed lens that matched: C (UX), E (contract & compat),
  //   G+H (operability & scale), S (security).
]

const FINDING_SCHEMA = {
  type: 'object',
  properties: {
    findings: { type: 'array', items: { type: 'object', properties: {
      file: { type: 'string' }, line: { type: 'number' },
      severity: { type: 'string' }, claim: { type: 'string' },
      proof: { type: 'string' }, proposedComment: { type: 'string' },
      chatExplanation: { type: 'string' }, codeSuggestion: { type: 'string' },
    }, required: ['file', 'line', 'severity', 'claim', 'proof'] } },
  }, required: ['findings'],
}

const found = await parallel(LENSES.map(l => () =>
  agent(`Read ${LENSES_DOC} and apply its section "${l.doc}" to this diff. Also
read the repo's instruction files (CLAUDE.md, AGENTS.md), the rules docs they
link, and the files in any rules directory the repo keeps (such as
.claude/rules/) that cover the surfaces you are reviewing. ${CTX}
${l.only ? `This is one shard of a large diff: report ONLY on these files:\n${l.only.join('\n')}` : ''}
Every finding carries file:line and a concrete mechanism as proof — a sequence,
not a concern. Drop anything you cannot prove from the code in front of you.`,
    { label: `find:${l.key}`, phase: 'Find', effort: EFFORT, schema: FINDING_SCHEMA })
))

const all = found.flatMap(f => f.findings)
const needsVerify = all.filter(f => /blocker|high/i.test(f.severity))

const verdicts = await parallel(needsVerify.map(f => () =>
  agent(`Adversarially REFUTE this finding. Assume it is wrong and try to prove
that from the code. ${CTX}
Finding: ${f.severity} — ${f.claim} (${f.file}:${f.line})
Proof offered: ${f.proof}`,
    { label: `verify:${f.file}:${f.line}`, phase: 'Verify', effort: EFFORT,
      schema: { type: 'object', properties: { isReal: { type: 'boolean' },
        reason: { type: 'string' } }, required: ['isReal', 'reason'] } })
    .then(v => ({ ...f, verdict: v }))
))

const critic = await agent(`Read ${LENSES_DOC} → "Completeness critic" and apply it. ${CTX}
Claims the finders made: ${all.map(f => `${f.severity}: ${f.claim}`).join('\n')}`,
  { label: 'critic', phase: 'Critic', effort: EFFORT })

return {
  findings: [...verdicts.filter(v => v.verdict?.isReal),
             ...all.filter(f => !/blocker|high/i.test(f.severity))],
  critic,
}
```

**An explicit `max` differs in exactly two ways** — run each Always lens group twice
independently and dedupe before verification, and make the refuter a 3-voter panel
(majority rules) on every Blocker/High. That pushes the agent count past the
default 15-agent guideline; the `max` tier is a deliberate choice to spend that,
so say the number out loud when you dispatch rather than quietly exceeding it.

`Workflow` runs in the background and notifies on completion — wait for the
notification, do not poll, and do not start stage D early.

## 4 · Write the state file

A later `/pr-review <N>` reads this round to decide it can run incrementally, so
the schema is not optional. Append it with the store's own script — never write
or edit the JSON by hand:

```bash
"$SK/state.py" append --owner <OWNER> --repo <REPO> --pr <N> --round-file <file>
```

It validates, refuses a duplicate finding id, stamps the timestamp and writes
atomically. The round's shape is at `$PRR/SKILL.md` Phase 7 step 5, with:

- `"kind": "self"`, `"effort": "<resolved level>"`, `"effortReason": "<the
  announcement's reason>"`, `"posted": false`
- `"heads"` — the `headRefOid` the review actually ran against
- `"verdicts"` — the deterministic verdict from `$PRR/references/scoring-rubric.md`,
  recorded for the record even though nothing is posted

The `high` path's subagent writes this itself. On the Workflow path the parent
writes it from the returned findings — one `Write` call, and it is the only place
the parent touches review output before triage.

Then remove the worktree: `"$SK/pr-context.sh" cleanup --ctx "$CTX"`.

## 5 · When the reviewer comes back empty

A clean review on a diff over ~150 lines is a signal to check, not to celebrate.
Confirm the reviewer actually saw the change — the returned severity counts should
account for a changed-file list matching `gh pr view <N> --json changedFiles`. A
reviewer that reviewed an empty range reports zero findings just as confidently as
one that reviewed a clean diff.
