---
name: jira-ticket-readiness
description: Use when the user wants to assess, triage, or check OPF Jira Bug and CDFailure tickets for required diagnostic information, then label ready tickets or comment on missing information. Uses the Atlassian MCP server and the fixed OPF component JQL. Keywords: Jira readiness, ticket quality, missing information, ready-for-work, needs-information, OPF bugs, CDFailure.
compatibility: Requires the shared Atlassian MCP server to be enabled and authenticated with permission to read, comment on, and edit OPF Jira issues.
---

# Jira Ticket Readiness

Assess unresolved OPF Bug and CDFailure tickets against the checklist below. Read evidence from the summary, description, labels, attachments, and all existing comments. Automatically update each assessed issue; do not ask for confirmation.

## Scope

Use this JQL exactly, including the readiness-label exclusion:

```jql
project = 'OPF'
AND component IN ("Cast Crew and Persona Service (CCP)", CIM-GO, "Content Builder", "Content Discovery Façade", "Content Delivery", "Content Discovery Gateway (CDG)", "Content Discovery Service  (CDS)", "Image Handler(IHS) (IHS)", "Image Service Lambda", "Image Metadata Server", "Metadata Server (MDS)", "Metadata Server Ingester", "Metamorph ", "Metamorph API", "OPUI Rails", "Recommendation Engine eXporter (REX)", "Recommendations Facade", "Search Façade", "Validation Service")
AND status NOT IN (Closed, Verified, Resolved)
AND resolution = Unresolved
AND type IN (Bug, CDFailure)
AND (labels IS EMPTY OR labels != "ready-for-work")
```

The explicit empty-label condition is required because `labels != "ready-for-work"` alone can omit issues with no labels.

Process every result, following pagination until no results remain. If the user explicitly asks for a dry run, perform the same assessment but make no Jira changes.

## Required Information

An issue is ready only when all four categories have usable, issue-specific evidence:

1. **Environment**: The deployment environment where the problem occurs, such as Production, Staging, or Development.
2. **Lab**: The lab where the problem occurs, or an explicit statement that no lab is involved or applicable.
3. **Description and reproduction**: A useful problem description and reproducible steps. Include relevant API calls, dataset details, and configuration changes when they apply. Explicit statements that a particular item is unchanged, unavailable, or not applicable count when reasonable. A Bruno or Postman script is helpful but not required.
4. **DEBUG logs**: DEBUG-level logs provided as text, attachment, or accessible link. An explicit, credible explanation that DEBUG logs cannot be obtained counts; a vague statement such as "see logs" does not.

Accept information wherever it appears in the summary, description, labels, attachments, or comments. Combine partial evidence across those sources. Do not require exact headings or template wording. Do not infer facts that are not stated, treat a component name as an environment or lab, or treat ordinary error text as DEBUG-level logs.

For CDFailure issues, CI/CD job output can support the reproduction and logs categories, but it must identify the failing operation and contain DEBUG-level detail or explicitly explain why DEBUG output is unavailable.

## Labels

Use these exact labels:

- `ready-for-work`: all required categories are present
- `needs-information`: one or more required categories are missing

The labels are mutually exclusive. A ready issue must have `ready-for-work` and must not have `needs-information`. An incomplete issue must have `needs-information` and must not have `ready-for-work`.

Preserve every unrelated existing label. Re-read the issue immediately before mutation and skip it if it is no longer unresolved or now has `ready-for-work`, preventing stale search results from overwriting concurrent updates.

## Missing-Information Comment

For an incomplete issue, add one concise comment containing only the missing categories and the standard marker shown below:

```text
Ticket readiness assessment: more information is needed.

Please add:
- Environment: identify where the issue occurs (for example, Production or Staging).
- Lab: identify the affected lab, or state that no lab applies.
- Description and reproduction: provide the problem details and reproducible steps, including relevant API calls, dataset, and configuration changes.
- DEBUG logs: attach, link, or paste DEBUG-level logs, or explain why they cannot be obtained.

[jira-ticket-readiness]
```

Include only bullets for categories actually missing. Keep the bullet wording above unchanged so repeated runs are deterministic.

Before commenting, inspect existing comments containing `[jira-ticket-readiness]`:

- If the newest such comment lists the same missing categories, do not add another comment.
- If the missing categories changed, add a new comment with the current list.
- Never edit or delete earlier comments.
- Never comment on a ready issue.

## Workflow

1. Verify that Atlassian MCP tools are available. If they are unavailable or unauthenticated, stop without changing tickets and explain that the shared `atlassian` MCP connection must be enabled, authenticated, and followed by an OpenCode restart.
2. Resolve the accessible Jira cloud/site, then run the fixed JQL and retrieve every page.
3. For each issue, retrieve the full summary, description, labels, attachments, and comments rather than assessing search-result snippets.
4. Record each category as present or missing with the exact evidence used. Keep this reasoning internal unless the user requests a dry run or detailed report.
5. Re-read the issue and perform the idempotency and concurrency checks.
6. If ready, add `ready-for-work` and remove `needs-information` in the smallest supported update.
7. If incomplete, add `needs-information`, remove `ready-for-work`, and add the standardized comment only when its current missing-category set is not already represented by the newest readiness comment.
8. Continue after an individual issue fails, recording the error without retrying a mutation whose outcome is uncertain.

Prefer one label update per issue. Never replace the complete labels array unless the Atlassian tool requires it; if it does, use the freshly read array and alter only the two readiness labels.

## Result

Return a concise summary:

```text
Assessed: <count>
Ready: <count>
Needs information: <count>
Unchanged: <count>
Failed: <count>
```

List issue keys under Ready, Needs information, and Failed. For failures, include the Jira error without exposing credentials or private authentication details.
