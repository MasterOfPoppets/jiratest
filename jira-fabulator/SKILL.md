---
name: jira-ticket-readiness
description: Assesses one Jira ticket for required diagnostic information, labels the ticket when ready, and comments on missing information. Use when triaging one Jira ticket, OPF bugs, CDFailure issues, or a ticket supplied by an automated job.
compatibility: Requires the shared Atlassian MCP server to be enabled and authenticated with permission to read, comment on, and edit OPF Jira issues.
---

# Jira Ticket Readiness

Assess exactly one Jira ticket supplied by the caller against the template below. Read evidence from the summary, description, labels, attachments, and all existing comments. Automatically update the assessed issue; do not ask for confirmation.

## Scope

The caller or cron job must supply exactly one Jira issue key. If zero keys or more than one key are supplied, stop and request exactly one Jira issue key. If the user explicitly asks for a dry run, perform the assessment but make no Jira changes.

## Required Information Template

Use each entry in this template as one required category. `missing_detail` is an instruction/template for an evidence-specific explanation, not a generic request:

```yaml
required_information:
  - name: Environment
    requirement: An explicit deployment environment where the problem occurs, such as Production, Staging, or Development. An environment implied only by the title or summary may be recorded as provided/implied, but remains missing until the caller confirms it explicitly.
    missing_detail: Explain which sources were checked and that the environment was only implied, absent, or otherwise insufficient; request explicit confirmation of the environment.
  - name: Lab
    requirement: An explicit lab where the problem occurs, or an explicit statement that no lab is involved or applicable.
    missing_detail: Cite the checked evidence and explain whether the lab is absent, ambiguous, or not explicitly declared; request the lab or an explicit not-applicable statement.
  - name: Description & Reproduction
    requirement: An actionable problem description and reproduction information that assesses the problem description, steps to reproduce, expected versus actual behavior, affected service or feature, relevant API calls, dataset, and configuration changes where applicable. Mentioning an action without describing how it was performed is insufficient. A Bruno or Postman script is optional.
    missing_detail: Identify the specific part(s) checked that are absent or non-actionable, explain why the available description cannot support assessment or reproduction, and request only the missing evidence.
  - name: DEBUG logs
    requirement: DEBUG-level logs supplied as an attachment, link, or pasted text, or a credible explanation of why DEBUG logs are unavailable.
    missing_detail: Cite the checked attachments, links, comments, or pasted content and explain why no DEBUG logs or credible unavailability explanation was sufficient; request the missing evidence or explanation.
```

An issue is ready only when every template entry has usable, issue-specific evidence. The template may be replaced or extended by configuration supplied with the assessment; when that occurs, evaluate and comment from the supplied entries using the same `name`, `requirement`, and `missing_detail` fields. Issue type and Priority are informational fields for the provided section; they are not readiness requirements.

Accept information wherever it appears in the summary, description, labels, attachments, or comments. Combine partial evidence across those sources. Do not require exact headings or template wording. Do not infer facts that are not stated, treat a component name as an environment or lab, or treat ordinary error text as DEBUG-level logs. Record whether evidence is explicit, implied, partial, attached, linked, or otherwise qualified. Every missing detail must cite what was checked and why the evidence is insufficient; never invent evidence.

For CDFailure issues, CI/CD job output can support the reproduction and logs categories, but it must identify the failing operation and contain DEBUG-level detail or explicitly explain why DEBUG output is unavailable.

## Labels

Use these exact labels:

- `ready-for-work`: all required categories are present
- `needs-information`: one or more required categories are missing

The analyzed issue must have an assessment label when processing completes: `ready-for-work` when ready or `needs-information` when incomplete. Add the applicable label if it is absent, even when the issue already has other labels.

Preserve all existing labels without exception except when an issue previously labelled `needs-information` is now ready. In that one case, remove only `needs-information` and add `ready-for-work`. Never remove, replace, rename, or overwrite any other label. In particular, assessing an incomplete issue must only add `needs-information`; it must not remove any existing label.

Re-read the issue immediately before mutation and skip it if it is no longer unresolved or now has `ready-for-work`, preventing stale search results from overwriting concurrent updates. A skipped issue with `ready-for-work` already satisfies the requirement that the assessed ticket carry an assessment label.

## Missing-Information Comment Template

For an incomplete issue, render this Markdown template. Include issue type and Priority in the provided section when Jira supplies them. Include every required category in the provided section when it has present, partial, or implied evidence; omit absent fields. A category may appear in both sections when evidence is implied but unconfirmed, especially Environment. Include items in the missing section only for unmet required categories.

```markdown
Ticket readiness assessment: more information is needed.

✅ What's been provided
{{#each provided_information}}
- {{name}}: {{evidence_detail}}
{{/each}}

❌ What's missing
{{#each missing_required_information}}
- {{name}}: {{missing_detail}}
{{/each}}

{{#if additional_comments}}
Additional comments: {{additional_comments}}
{{/if}}

[jira-ticket-readiness]
```

For `provided_information`, render evidence-derived details that distinguish explicit, implied, partial, attached, linked, or other applicable status and cite the useful evidence. For `missing_required_information`, render the category's evidence-specific `missing_detail`, including what was checked and why it was insufficient. Optionally set `additional_comments` to one or two concise sentences when useful ticket-specific context does not fit either list; omit the field and its heading when there is nothing material to add. Do not repeat list content, use generic filler, or invent evidence.

Before commenting, inspect existing comments containing `[jira-ticket-readiness]`:

- Compare the current missing category set, normalized evidence-derived details, and any material additional comments against the newest marker comment.
- If the category set, normalized evidence-derived details, and material additional comments are equivalent, do not add another comment.
- If categories or material evidence/details changed, add a new comment with the current assessment.
- Never edit or delete earlier comments.
- Never comment on a ready issue.

## Workflow

1. Verify that Atlassian MCP tools are available. If they are unavailable or unauthenticated, stop without changing the ticket and explain that the shared `atlassian` MCP connection must be enabled, authenticated, and followed by an OpenCode restart.
2. Resolve the accessible Jira cloud/site and retrieve the one issue supplied by the caller.
3. Retrieve the full summary, description, issue type, Priority, labels, attachments, and comments for that issue.
4. Record provided evidence and a missing reason for each required category, using only evidence found in the issue. Render the comment template if any required category is incomplete.
5. Re-read the issue and perform the idempotency and concurrency checks, including comparison of the current missing category set and normalized evidence-derived details with the newest readiness comment.
6. If ready, add `ready-for-work`. If and only if `needs-information` is already present, remove that label while preserving every other label.
7. If incomplete, add `needs-information` if absent, preserve every existing label, and render the comment only when its current category set or material evidence/details are not already represented by the newest readiness comment.

Prefer additive label operations. Never replace the complete labels array unless the Atlassian tool requires it; if it does, construct the update from the freshly read array, preserve every value, add the applicable assessment label, and remove only `needs-information` when changing that assessment to `ready-for-work`. Verify the resulting labels after mutation; if the applicable assessment label is absent, treat the issue as failed.

