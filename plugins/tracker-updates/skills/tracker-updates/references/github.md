# GitHub mechanics

Everything goes through `gh`. Resolve `<owner>/<repo>` from the pasted URL, else
from the current checkout (`gh repo view --json nameWithOwner`).

## Issues

- **Find:** `gh issue list -R <owner>/<repo> --search "<words>" --state all`. Search
  before creating.
- **Create:** `gh issue create -R <owner>/<repo> --title "…" --body-file <file>`,
  plus `--label`, `--milestone`, `--project "<title>"` when the repo uses them.
  Write the body to a file so newlines and backticks survive.
- **Edit:** `gh issue edit <n> -R <owner>/<repo> --body-file <file>` (or `--title`).
- **Comment:** `gh issue comment <n> -R <owner>/<repo> --body-file <file>`.

## Project status updates (Projects v2)

Projects need the `project` token scope. If `gh auth status` doesn't list it, ask
the user to run `gh auth refresh -s project` — don't run it for them.

1. **Project id:** `gh project view <number> --owner <owner> --format json --jq .id`
   (the number is in the project URL: `/projects/<number>`).
2. **Read the last 3 updates:**
   ```bash
   gh api graphql -f query='query($id:ID!){node(id:$id){... on ProjectV2{
     statusUpdates(last:3){nodes{id body status createdAt}}}}}' -f id=<project id>
   ```
3. **Post a new update:**
   ```bash
   gh api graphql -f query='mutation($id:ID!,$body:String!,$status:ProjectV2StatusUpdateStatus!){
     createProjectV2StatusUpdate(input:{projectId:$id,body:$body,status:$status}){
       statusUpdate{id}}}' -f id=<project id> -F body=@<file> -f status=ON_TRACK
   ```
   Optional `startDate` / `targetDate` (`YYYY-MM-DD`). To edit an existing update
   instead, use `updateProjectV2StatusUpdate` with `statusUpdateId`.
4. **Health:** `status` is `ON_TRACK` | `AT_RISK` | `OFF_TRACK` (also `COMPLETE`,
   `INACTIVE` for a finished or paused project).

**No project:** post the update as a comment on the tracking issue (the one the
user names, or the open issue labelled as the epic), with the health call in the
first bullet: `**On track.**`, `**At risk:** <why>`, `**Off track:** <why>`.
