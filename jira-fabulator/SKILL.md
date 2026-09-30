---
name: jira-ticket-readiness
description: Assesses one Jira ticket for required diagnostic information, applies a readiness label, and comments on missing information. Use when triaging a Jira ticket, OPF bug, CDFailure issue, or ticket supplied by an automated job.
compatibility: Requires the shared Atlassian MCP server to be enabled and authenticated with permission to read, comment on, and edit an OPF Jira issue.
---

# Jira Ticket Readiness

Assess the Jira ticket supplied by the caller against the template below. Read evidence from its summary, description, labels, attachments, and all existing comments. Automatically update the ticket; do not ask for confirmation.

## Scope

The caller or cron job supplies exactly one Jira issue key. If no issue key or more than one issue key is supplied, stop and request exactly one. If the user explicitly asks for a dry run, perform the same assessment but make no Jira changes.

## Required Information Template

Use each entry in this template as one required category:

```yaml
required_information:
  - name: Environment
    requirement: The deployment environment where the problem occurs, such as Production, Staging, or Development.
    missing_prompt: Identify where the issue occurs, for example Production or Staging.
  - name: Lab
    requirement: The lab where the problem occurs, or an explicit statement that no lab is involved or applicable.
    missing_prompt: Identify the affected lab, or state that no lab applies.
  - name: Description and reproduction
    requirement: A useful problem description and reproducible steps, including relevant API calls, dataset details, and configuration changes when they apply. Explicit statements that an item is unchanged, unavailable, or not applicable count when reasonable. A Bruno or Postman script is helpful but not required.
    missing_prompt: Provide the problem details and reproducible steps, including relevant API calls, dataset, and configuration changes.
  - name: DEBUG logs
    requirement: DEBUG-level logs provided as text, attachment, or accessible link. An explicit, credible explanation that DEBUG logs cannot be obtained counts; a vague statement such as "see logs" does not.
    missing_prompt: Attach, link, or paste DEBUG-level logs, or explain why they cannot be obtained.
```

An issue is ready only when every template entry has usable, issue-specific evidence. The template may be replaced or extended by configuration supplied with the assessment; when that occurs, evaluate and comment from the supplied entries using the same `name`, `requirement`, and `missing_prompt` fields.

Accept information wherever it appears in the summary, description, labels, attachments, or comments. Combine partial evidence across those sources. Do not require exact headings or template wording. Do not infer facts that are not stated, treat a component name as an environment or lab, or treat ordinary error text as DEBUG-level logs.

For CDFailure issues, CI/CD job output can support the reproduction and logs categories, but it must identify the failing operation and contain DEBUG-level detail or explicitly explain why DEBUG output is unavailable.

## Labels

Use these exact labels:

- `ready-for-work`: all required categories are present
- `needs-information`: one or more required categories are missing

The analyzed issue must have an assessment label when processing completes: `ready-for-work` when ready or `needs-information` when incomplete. Add the applicable label if it is absent, even when the issue already has other labels.

Preserve all existing labels without exception except when an issue previously labelled `needs-information` is now ready. In that one case, remove only `needs-information` and add `ready-for-work`. Never remove, replace, rename, or overwrite any other label. In particular, assessing an incomplete issue must only add `needs-information`; it must not remove any existing label.

Re-read the issue immediately before mutation and skip it if it is no longer unresolved or now has `ready-for-work`, preventing stale search results from overwriting concurrent updates. A skipped issue with `ready-for-work` already satisfies the requirement that analyzed tickets carry an assessment label.

## Missing-Information Comment Template

For an incomplete issue, render a comment from the missing required-information entries:

```text
Ticket readiness assessment: more information is needed.

Please add:
{{#each missing_required_information}}
- {{name}}: {{missing_prompt}}
{{/each}}

[jira-ticket-readiness]
```

Render one bullet per missing category and no bullets for categories that are present. Preserve the configured `name` and `missing_prompt` text so repeated runs are deterministic.

Before commenting, inspect existing comments containing `[jira-ticket-readiness]`:

- If the newest such comment lists the same missing categories, do not add another comment.
- If the missing categories changed, add a new comment with the current list.
- Never edit or delete earlier comments.
- Never comment on a ready issue.

## Workflow

1. Verify that Atlassian MCP tools are available. If they are unavailable or unauthenticated, stop without changing tickets and explain that the shared `atlassian` MCP connection must be enabled, authenticated, and followed by an OpenCode restart.
2. Resolve the accessible Jira cloud/site and retrieve the issue supplied by the caller.
3. Retrieve the issue's full summary, description, labels, attachments, and comments.
4. Record each category as present or missing with the exact evidence used. Keep this reasoning internal unless the user requests a dry run or detailed report.
5. Re-read the issue and perform the idempotency and concurrency checks.
6. If ready, add `ready-for-work`. If and only if `needs-information` is already present, remove that label while preserving every other label.
7. If incomplete, add `needs-information` if absent, preserve every existing label, and render the comment template only when its current missing-category set is not already represented by the newest readiness comment.
8. If the assessment fails, report the error without retrying a mutation whose outcome is uncertain.

Prefer additive label operations. Never replace the complete labels array unless the Atlassian tool requires it; if it does, construct the update from the freshly read array, preserve every value, add the applicable assessment label, and remove only `needs-information` when changing that assessment to `ready-for-work`. Verify the resulting labels after mutation; if the applicable assessment label is absent, treat the issue as failed.

