# The hunt: 17 patterns

Every pattern below is a real finding on a real PR in one production monorepo —
1–12 from human reviewers, 13–17 from review bots. The bot ones are not nitpicks:
across 13 PRs the author accepted and fixed 50 of 78 bot findings and rejected 2,
so treat them as findings caught one round too late. Identifiers are renamed; the
mechanisms are exactly as found. Walk all seventeen against your own diff. For
each: write why it cannot apply, or write the finding.

A finding needs a concrete sequence. *"There may be a race"* is not a finding.
*"A reads status=running, awaits the upload; B sets status=failed; A resumes and
writes status=running"* is.

---

## Lifecycle & concurrency

### 1 · Shared flag, per-request need
A module-level variable or a component `ref` is read by concurrent attempts, so
attempt #2 sees attempt #1's state.

> `dismissedWhileStartingRef` belongs to the component, so every start request reads the same flag. Close-and-reopen resets it while the first request is still pending; the stale response then sets `activeSessionIdRef`.
> *Impact:* a remote browser session becomes unreachable and keeps consuming capacity.
> *Fix:* per-attempt id; invalidate on close/back/unmount; after the await, re-check the attempt id and cancel the returned session if it is stale.

**Ask:** is this state per-component/per-module, or does it need to be per-request?
What resets it, and can that happen while a request is in flight?

### 2 · Fire-and-forget teardown
Cleanup is started but not awaited, so the next start races the old stop.

> `releaseSession()` starts `cancelRecordingClient` and returns immediately. `startBrowserSession` then calls `startRecordingClient` while the old stop is still in flight — and can hit the very 429 the PR was fixing.
> *Fix:* return the cancellation promise and await it in the retry path; exit handlers may still `void` it. Test with the cancellation held pending.

**Ask:** does every path that starts something new await the teardown of the old one?

### 3 · Cleared-before-settled handle
The handle is nulled before its async operation completes, so a guard reads
"already clean" and skips the wait.

> `releaseSession` clears `activeSessionIdRef` before the cancel request settles. Reopen during that window hits the early return and treats teardown as complete, because the pending promise was never stored.
> *Fix:* track cancellation promises by session id; await pending teardown before any start; re-check the attempt id immediately after the await.

**Ask:** between "I cleared the handle" and "the operation finished", what does a
concurrent caller see?

### 4 · Lease without a fencing token
A claim is identified by row + status rather than by owner, so two holders can
both believe they own it.

> The renewal updates by session id and status. If renewals stall past 120s, request B reclaims the row; A later resumes and updates the same timestamp. Both continue as owners — both build an artifact and upload it to object storage, and the last `setCompleted` detaches the other's artifact.
> *Fix:* store an attempt token on claim; require it for renewals and terminal writes; stop when a renewal updates zero rows; re-check ownership before the side effect.

**Ask:** if my `WHERE` clause does not name *me*, what stops someone else's write
from satisfying it?

### 5 · Read-modify-write across an await
Status is read, remote work is awaited, then a write lands based on the stale read.

> `cancelActiveSession` sets the row to `failed`, but Begin and the first event upload both read the old status, await remote work, then write `"running"` — restoring a cancelled row and clobbering Complete's `completing` claim.
> *Fix:* conditional transition (`queued` → `running`) instead of an unconditional write; drop the redundant second update; discard a late Begin result after Cancel. Race tests both ways.

**Ask:** for every `await` between reading state and writing it, who else can write
in that window? Is the write conditional on the state I read?

---

## Contract width

### 6 · Field with a wider contract
You write a column for your local meaning; other subsystems already gate on it.

> `disabled_at` is not trigger-local. `filterRunnableAccounts` maps any non-null `disabled_at` to `reason: "disabled"`, and both schedule lanes skip the job; the legacy lane feeds `trackMissingAccountFailure`, accruing toward auto-pause (20 failures, or 5 over 5 days). An account missing one *trigger* scope therefore loses every scheduled job bound to it.
> *Fix:* either a signal that stops polling without failing the schedule gate, or state the consequence in the PR description **and** the user-facing docs.

**Ask:** `grep -rn "<column>"` — every reader. Does my new meaning hold for all of them?

### 7 · Second writer bypasses the new gate
You gated one path; another path still writes unconditionally.

> The new gate is correct, but `AccountRepository.updateAccountTokens` clears `disabled_at`/`disabled_reason` **unconditionally**, and its caller then resolves the reconnect notification — silently re-enabling the account and restarting the 403 loop with the alarm switched off. Dead today, but nothing pins the invariant, and a dead-code linter will not catch an unused class method.
> *Fix:* mirror the gate in the repository, or delete the dead method.

**Ask:** `grep` every writer of the field, not just the one I changed. Which ones
skip my new condition?

### 8 · Shape assumed from one provider
A parser or classifier is built from one vendor's payload; siblings emit another
shape and fall through silently.

> `typeof status === "string"` never fires: `GaxiosError` sets `status` from `response.status`, a **number**. Only the legacy `errors[]` arm is live. Gmail v1 emits `errors[]` and is genuinely fixed; `driveactivity` v2 and Sheets v4 are One Platform (`details[]`) and fall through to 502, keeping the retry loop. Separately, Outlook/Teams/SharePoint funnel everything into `UpstreamHttpError(status, message)`, so the Graph code survives only inside interpolated text — no classification at all.
> *Fix:* read the reason from `error.response?.data?.error?.details?.[].reason`; carry the Graph code on `UpstreamHttpError`; or scope the doc comment to Google-only and say so in the docs.

**Ask:** which other implementations reach this code? Reconstruct one real payload
per provider and run the function against each.

### 9 · Fixture-shaped tests
Tests hand-build the payload the code already expects, so they cannot fail.

> `is-upstream-scope-error.test.ts` asserts `{code: 403, status: "ACCESS_TOKEN_SCOPE_INSUFFICIENT"}` — a shape no client produces. The Drive fixture hands the path `errors:[{reason:"insufficientPermissions"}]`, the shape that already works.
> *Fix:* capture one real error payload per provider and fixture from that.

**Ask:** did I write the fixture from the code, or from a captured real payload?
A test whose input I derived from my own implementation tests nothing.

### 10 · Type completeness mistaken for correctness
`Record<K, string>` forces the key to exist; it does not force the value to be right.

> Nothing catches the copy bug because no test covers `insufficient_scopes` — `Record<ReconnectReason, string>` forces the key to exist, not the sentence to read.

**Ask:** where am I relying on exhaustiveness to mean correctness?

---

## Surface semantics

### 11 · Global semantics flipped for one nested case
A prop flipped to fix a nested interaction removes a guarantee everywhere that
component is used.

> Setting the Add Credential dialog non-modal so a nested authenticator can scroll means Radix renders no `Dialog.Overlay`, does not trap focus, and does not block outside pointer events — so `showOverlay` produces no backdrop and controls behind can still be clicked. The same change is in `edit-credential-dialog.tsx`; opened from `ManageCredentialsDialog`, the outer modal remains the active `react-remove-scroll` lock and the Add/Edit portal sits outside it, so the lock can reject wheel events on long forms.
> *Impact:* a click behind the dialog closes the parent popover and unmounts the form, losing entered values; on a short viewport the form cannot be scrolled.
> *Fix:* keep each dialog modal while it is the top layer; flip the parent only while the child is open, or mount the child inside the parent's scroll lock. Integration tests from both entry points.

**Ask:** what does this flag guarantee today, and which of those guarantees does
every *other* caller depend on?

### 12 · Composed copy
A string that reads fine standalone breaks when composed with a suffix.

> "…but it hasn't been given every permission it needs **since 21 August**" reads as a permission that lapsed on that date — the exact misreading the clause exists to prevent. The other two variants end in a state that `since` modifies cleanly; a negated perfect does not. Fix: *"…but it has been missing a permission it needs"*. It ships in the reconnect email and Slack message.

**Ask:** where does this string get concatenated? Read every composed result aloud,
not just the literal I edited.

---

## Bounds & shared tables

### 13 · Unbounded multi-row query
A repeating pass selects every matching row, so its cost scales with the backlog
rather than with the tick.

> `getStaleBrowserSessions()` returns every matching row with no SQL-level `LIMIT`, so a large cleanup backlog lands in one tick. A second cleanup query in the same repository had the same shape: each cron tick transferred every matching terminal row.
> *Fix:* a SQL `LIMIT` on the query, sized to the tick, and let the next tick take the rest.

**Ask:** does every new query have a bound *in SQL*? A cron/sweep query with no
`LIMIT` is a bug even when today's table is small — and slicing in memory after
the fetch does not count.

### 14 · Predicate too broad inside a shared table
The table holds more than one kind of row. Your `WHERE` names the status but not
the kind, so it matches somebody else's row.

> `jobs` holds both `export` and `import` rows. `claimExportJob` matched an `import` row sitting in `queued` state and flipped it to `running`. The cancellation `UPDATE` filtered only on status and `deleted_at`, so it could cancel an import.
> *Fix:* add the discriminator — `kind = 'export'` — to every claim, cancel and sweep predicate on that table.

**Ask:** what else lives in this table? List the discriminator columns (`provider`,
`kind`, `type`, `source`) and check that every predicate I wrote names the one
that scopes it to my rows. Then check `deleted_at` and the tenant scope too.

### 15 · Whole-object write clobbers a sibling field
You write one key of a JSONB/object column; a later write replaces the whole
object and drops it.

> The route writes `metadata.releasedAt`, but the `updateMetadata` call further down replaces the complete `metadata` JSONB — so the field it just set is gone.
> *Fix:* merge rather than replace, or order the writes so the last one carries every key.

**Ask:** for every JSONB / object / spread write in the path — does a later write
in the same request replace the whole value? Read the writes in execution order,
not in file order.

## Hygiene the bots will find if you don't

### 16 · Protocol value as a bare literal
A string or number that is part of a contract — a config protocol, a metric label,
a shared window — appears as a literal, twice.

> `"false"` is a configuration protocol value, compared inline. `"ok"` and `"error"` define one Prometheus label contract, written at two sites. `35 * 60 * 1000` is computed independently in the stop pass and the pass whose docblock claims it matches.
> *Fix:* a named constant or a string-literal union, defined once, used at every site — including the pre-seeded series.

**Ask:** does this literal appear anywhere else, or does anything outside this file
have to agree with it? A metric label and a config value are contracts; a
duplicated timing window is a future divergence. (Metric name and label changes
are breaking for dashboards and alerts.)

### 17 · Test-double hygiene
Two specific failures, both caught repeatedly:

> **Incomplete reset** — the `beforeEach` clears only `counterIncMock`, leaving the other shared module-level mocks holding calls and implementations across tests. Order-dependent green.
> **Interface bypassed** — the logger and repository mocks use `unknown` double assertions instead of precise mock types, so the double is free to drift from the interface it stands in for.
> *Fix:* reset every shared mock (`vi.resetAllMocks()` in vitest, or your framework's equivalent); type the double against the real interface so a signature change fails the test file.

**Ask:** does every module-level mock get reset? Would my double still compile if
the real interface changed?

---

## Closing the hunt

Findings go in this format, with `file:line` for every claim:

```
Bug: <one line>
<mechanism — cite the code that makes it true>
Impact: <what a user or operator sees>
Fix: <smallest change, plus the test that pins it>
```

Then: fix it, or get an explicit written waiver — and put the waived finding in the
PR description.
