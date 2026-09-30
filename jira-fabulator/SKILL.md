---
name: jira-ticket-readiness
description: Assesses one Jira ticket for required diagnostic information, labels the ticket when ready, and comments on missing information. Use when triaging one Jira ticket, OPF bugs, CDFailure issues, or a ticket supplied by an automated job.
compatibility: Requires the shared Atlassian MCP server to be enabled and authenticated with permission to read, comment on, and edit OPF Jira issues.
---

# Jira Ticket Readiness

Assess exactly one Jira ticket supplied by the caller against the template below. Read all named evidence sources, classify every category, automatically update the assessed issue, and do not ask for confirmation.

## Scope

The caller or cron job must supply exactly one Jira issue key. If zero keys or more than one key are supplied, stop and request exactly one Jira issue key. If the user explicitly asks for a dry run, perform the assessment but make no Jira changes.

## Evidence model

Classify each required category in exactly one state:

- **confirmed/provided**: explicit, internally consistent evidence; satisfies the requirement.
- **inferred/ambiguous (amber)**: a plausible value inferred from one or more named sources, but not explicit or conflicting; does not satisfy the requirement and requires clarification.
- **missing**: no credible evidence; does not satisfy the requirement.

A ticket is ready only when every required category is confirmed/provided. Any amber or missing category requires `needs-information`.

Retrieve and inspect these Jira fields as evidence sources: **Environment**, **labels**, **summary**, **Customer**, and **Detected in**. Also inspect the description, attachments, and all comments. Customer and Detected in may be custom fields with instance-specific IDs: discover and read them by display name from Jira field metadata; never guess field IDs.

Use the sources to infer candidate Environment, Lab, and Customer values, without turning inference into fact. Environment names may occur in summary, labels, Environment, or Detected in; lab names or identifiers may occur in summary, labels, Environment, or Detected in; customer names may occur in Customer, summary, labels, description, or comments. For every inference, cite the exact signal and source. Do not infer from unrelated tokens. If evidence conflicts, list each candidate and its source without choosing.

## Required Information Template

Use each entry as one required category. `missing_detail` is an evidence-specific explanation, not a generic request:

```yaml
required_information:
  - name: Environment
    requirement: An explicit deployment environment where the problem occurs, such as Production, Staging, or Development. A plausible environment inferred from summary, labels, Environment, or Detected in may be shown amber, but never satisfies readiness.
    missing_detail: Cite the checked sources and exact signal, explain whether the environment is missing, ambiguous, conflicting, or only inferred, and request explicit confirmation.
  - name: Lab
    requirement: An explicit lab where the problem occurs, or an explicit statement that no lab is involved or applicable. A plausible lab inferred from summary, labels, Environment, or Detected in may be shown amber, but never satisfies readiness.
    missing_detail: Cite the checked sources and exact signal, explain whether the lab is missing, ambiguous, conflicting, or only inferred, and request the lab or an explicit not-applicable statement.
  - name: Customer
    requirement: An explicit Customer field/value, or an explicit statement in ticket content identifying the customer or saying none/not applicable. An inferred customer is amber and incomplete.
    missing_detail: Cite the checked Customer field, summary, labels, description, and comments, explain whether the customer is absent, ambiguous, conflicting, or only inferred, and request an explicit customer or none/not-applicable statement.
  - name: Description & Reproduction
    requirement: An actionable problem description and reproduction information that assesses the problem description, steps to reproduce, expected versus actual behavior, affected service or feature, relevant API calls, dataset, and configuration changes where applicable. Mentioning an action without describing how it was performed is insufficient. A Bruno or Postman script is optional.
    missing_detail: Identify the specific part(s) checked that are absent or non-actionable, explain why the available description cannot support assessment or reproduction, and request only the missing evidence.
  - name: DEBUG logs
    requirement: DEBUG-level logs supplied as an attachment, link, or pasted text, or a credible explanation of why DEBUG logs are unavailable.
    missing_detail: Cite the checked attachments, links, comments, or pasted content and explain why no DEBUG logs or credible unavailability explanation was sufficient; request the missing evidence or explanation.
```

The template may be replaced or extended by configuration supplied with the assessment; evaluate supplied entries using the same `name`, `requirement`, and `missing_detail` fields. Issue type and Priority are informational confirmed values when present, not readiness requirements. Combine explicit evidence across sources, but never promote partial or implied evidence to confirmed/provided. Do not require exact headings or template wording, treat a component name as an environment or lab, or treat ordinary error text as DEBUG-level logs. For CDFailure issues, CI/CD job output can support reproduction and logs when it identifies the failing operation and contains DEBUG-level detail or explicitly explains why DEBUG output is unavailable.

## Labels

Use these exact labels:

- `ready-for-work`: all required categories are confirmed/provided
- `needs-information`: one or more required categories are amber or missing

The analyzed issue must have the applicable assessment label when processing completes. Preserve all existing labels without exception except when an issue previously labelled `needs-information` is now ready: remove only `needs-information` and add `ready-for-work`. Never remove, replace, rename, or overwrite any other label.

Re-read the issue immediately before mutation and skip it if it is no longer unresolved or now has `ready-for-work`, preventing stale search results from overwriting concurrent updates.

## Missing-Information Comment Template

For an incomplete issue, render this Markdown template. Omit every empty section. A category must appear in exactly one of the three state sections. Include Issue type and Priority in confirmed information when Jira supplies them.

```markdown
Ticket readiness assessment: more information is needed.

{{#if confirmed_information}}
✅ What's been provided
{{#each confirmed_information}}
- {{name}}: {{evidence_detail}}
{{/each}}
{{/if}}

{{#if inferred_information}}
🟠 What's inferred — please confirm
{{#each inferred_information}}
- {{name}}: Candidate: {{inferred_value}}; evidence: {{evidence_source}}; please confirm: {{clarification_needed}}
{{/each}}
{{/if}}

{{#if missing_required_information}}
❌ What's missing
{{#each missing_required_information}}
- {{name}}: {{missing_detail}}
{{/each}}
{{/if}}

{{#if additional_comments}}
Additional comments: {{additional_comments}}
{{/if}}

[jira-ticket-readiness]
```

`inferred_information` entries must include `name`, `inferred_value`, `evidence_source`, and `clarification_needed`; clearly state the candidate, exact source/signal, and requested confirmation. Conflicting evidence must show candidates and sources without choosing. `missing_required_information` must cite what was checked and why it was insufficient. Never invent evidence, repeat list content, or use generic filler.

## Ready Confirmation Comment Template

For a ticket with every required category confirmed/provided, after the `ready-for-work` label update has been verified, render:

```markdown
✅ This ticket has the information needed and is ready for work. Thanks for providing the required detail.

[jira-ticket-readiness]
```

## Idempotency

Inspect existing comments containing `[jira-ticket-readiness]` before posting. Compare the state/category sets, normalized confirmed evidence details, normalized inferences (including candidate values, exact sources/signals, and clarification requests), and material additional comments against the newest marker comment. Equivalent data means no duplicate comment; any category/state change or materially changed evidence or inference produces a new assessment comment. Never edit or delete earlier comments. Do not post an incomplete comment for a ready issue; an older incomplete marker does not count as the ready confirmation.

## Workflow

1. Verify Atlassian MCP tools are available and authenticated; otherwise stop without changing the ticket and explain that the shared `atlassian` MCP connection must be enabled, authenticated, and followed by an OpenCode restart.
2. Resolve the accessible Jira cloud/site and retrieve exactly one issue supplied by the caller.
3. Retrieve full summary, description, issue type, Priority, labels, attachments, comments, and the Environment, Customer, and Detected in fields discovered by display name from Jira field metadata.
4. Classify every required category as confirmed/provided, inferred/ambiguous, or missing. Record exact evidence sources/signals, candidates and conflicts, and evidence-specific missing or clarification instructions. Render the incomplete template when any category is amber or missing.
5. Re-read the issue immediately before mutation; perform concurrency and idempotency checks using state/category sets, normalized evidence details/inferences, and additional comments.
6. If all required categories are confirmed, add `ready-for-work`; only if `needs-information` is present, remove that label while preserving every other label. Verify `ready-for-work` before posting the ready confirmation, unless an equivalent ready marker already exists.
7. If incomplete, add `needs-information` if absent, preserve every existing label, and post the current three-state assessment only when it is not already represented by the newest marker comment. Never post it for a ready issue.

Prefer additive label operations. If the Atlassian tool requires replacing the labels array, construct it from the freshly read array, preserve every value, add the applicable assessment label, and remove only `needs-information` when changing to `ready-for-work`. Verify the resulting labels; if the applicable assessment label is absent, treat the issue as failed.

