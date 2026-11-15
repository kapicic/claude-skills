# Jira mechanics

Jira goes through the Atlassian MCP (the claude.ai Atlassian connector, or a local
Atlassian MCP server). Load its tools with `ToolSearch "+atlassian jira"`. Tool names
below are the Atlassian Remote MCP's; **read each loaded tool's schema before the
first call** — parameter names and the description format (Markdown vs. Atlassian
Document Format) differ between server versions.

- **Site:** most calls need a `cloudId`. Get it once from
  `getAccessibleAtlassianResources`; if the user has several sites, match the host
  of the pasted URL.
- **Find:** `searchJiraIssuesUsingJql`, e.g.
  `project = ENG AND text ~ "SCIM" ORDER BY updated DESC`. Search before creating.
- **Read:** `getJiraIssue` with the key (`ENG-123`).
- **Create:** `createJiraIssue` needs the project key, the issue type (`Task`, `Bug`,
  `Story`, `Epic`), and the summary. Check the project's types and required fields
  with `getJiraProjectIssueTypesMetadata` / `getJiraIssueTypeMetaWithFields` when a
  create is rejected.
- **Edit:** `editJiraIssue` with only the fields that change.
- **Comment:** `addCommentToJiraIssue`.
- **Status:** leave it alone unless told; moving it is `transitionJiraIssue` with a
  transition id from `getTransitionsForJiraIssue`.

## Project status updates

Jira has no project status update object. Post the update as a **comment on the
epic** (or the parent issue the project hangs off), and ask which epic when it isn't
obvious from the link.

**Health:** Jira has no health field. Put the call in the first bullet:
`**On track.**`, `**At risk:** <why>`, or `**Off track:** <why>`. Read the epic's
last 3 comments first to find the previous call and carry it forward.

If the team keeps status in Confluence instead, the Atlassian MCP can write it there
(`createConfluencePage` / `updateConfluencePage`) — only when the user says so.
