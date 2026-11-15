# Blast radius (phase 2)

The goal: know, before writing a line, everything outside the diff that your change
reaches. This is the phase that decides whether the PR takes one review round or three.

## The table

Build it for every piece of mutable state the change touches — DB column, row
status, React ref, module-level variable, cache entry, lease, in-flight promise.

| State | Who owns it | Other writers | Readers that gate on it | Concurrent access |
|---|---|---|---|---|
| `accounts.disabled_at` | token service | `updateAccountTokens` (unconditional) | `filterRunnableAccounts` → both schedule lanes → auto-pause | one poll per minute per account |

Empty cells are the answer you are looking for — an empty "other writers" column
that turns out to be wrong is exactly finding #7.

## The three questions

### 1 · Widening
> Does the field, flag, or status I am setting already mean something to code I am
> not touching?

```bash
# every reference to a column, across services
rg -n 'disabled_at' --glob '!node_modules'
# checked-in schema or generated types show the table's real shape without probing the DB
rg -n 'disabled_at' <schema file or generated types>
```

Read every *reader*, not just the writers. A column's meaning is defined by who
gates on it. If your new meaning does not hold for one of them, you have either a
bug or a PR-description obligation — decide which, in phase 3, not in review.

### 2 · Siblings
> How many implementations of this shape exist, and does my change cover each one?

```bash
# call sites of the function you changed
rg -n 'isUpstreamScopeError' --glob '!node_modules'
# other implementations of the same interface / provider family
ls src/oauth/providers/
rg -n 'UpstreamHttpError' --glob '!node_modules'
```

For a parser, classifier, or selector: **reconstruct one real payload per provider
and run the function against each.** Write the result matrix into the plan:

```
gmail v1 / drive v3 legacy 403 : true
driveactivity v2 / sheets v4   : false   <- unclassified
graph (outlook/teams/sharepoint): false  <- no code carried at all
```

A matrix with a `false` in it is either scope you are taking on, or a documented
limitation. Both are fine; silence is not.

### 3 · Interleaving
> For each `await` in the path, what can happen while it is pending?

Walk the path and list the awaits. For each one, name the other actors that can
touch the same state in that window: a second request, a retry, a cancel, an
unmount, a lease renewal, a cron tick.

```
startBrowserSession
  ├─ await releaseSession()      ← returns before the stop completes; retry can start here
  ├─ await startRecordingClient  ← dialog may have closed; attempt id may be stale
  └─ set activeSessionIdRef      ← a later response can overwrite this
```

Then, for each interleaving, decide: impossible (say why), handled (say how), or a
bug (fix it in this PR or explicitly out of scope).

## Layer-specific probes

- **DB column** — the checked-in schema or generated types are the reference; read
  them instead of probing `information_schema`. Check the soft-delete contract
  (`deleted_at` or equivalent) and how the table is scoped to a tenant or owner.
- **Route** — is it public? Then whatever the repo keeps in step with its public
  API (a CLI command, an SDK, the spec, a version) changes in the same PR. Does it
  need an auth, permission or entitlement gate?
- **Repository / query** — bounded query? N+1? Any in-memory filter/sort/paginate
  after the fetch is a bug.
- **List endpoint** — if it caps or paginates, the client must paginate and filter
  server-side.
- **Server-side fetch of a caller-controlled URL** — must go through the repo's
  SSRF guard.
- **Metric** — defined once; a name or label change is breaking for dashboards and
  alert policies.
- **New protected route or page** — find every allowlist that enumerates routes
  (deep links, redirects, auth middleware, CSP) and add it where it belongs.
- **Migration** — its own bottom PR; build an index on an existing table without
  locking it (`CREATE INDEX CONCURRENTLY` in Postgres); regenerated types on the
  *same* branch.
- **React state** — per-component or per-request? What happens on unmount, on
  close-and-reopen, on a second click while the first is pending?
- **Cross-cutting** — match the touched paths against the repo's invariant, rules
  or architecture docs and read every one that applies.

The overlay's `Map` section adds the repo's own probes (registries every new table
must join, files every new route must touch).

## What this produces

Three written answers and one table, presented in phase 3 alongside the plan. The
widening answer and the sibling matrix go on to become paragraphs in the PR
description (phase 7) — that is where they stop being review comments.
