# PM Adapter: Jira

Story ID format: variable prefix — full key required (e.g., `PROJ-123`, `ENG-456`).

## Session Start

At the beginning of each session using the Jira adapter, ask the user:

> "What is the Jira issue key for this session? (e.g., PROJ-123)"

Store the key for all subsequent PM operations in this session.

## Fetch Story

Use the `jira` CLI (Ankitpokhrel):

```bash
jira issue view {KEY} --plain
```

Parse from output:
- **Title/Summary**: first line after the key
- **Description**: body text
- **Acceptance Criteria**: look in two places — a dedicated custom field (often labeled "Acceptance Criteria") and as a structured section within the description (e.g., a heading or bullet list). Surface whatever is found in either location.
- **Issue type**: type field (Bug, Story, Task, etc.)
- **Status**: current workflow state
- **Comments**: comment thread at the bottom of output

## Post Comment

```bash
jira issue comment add {KEY} --body "{text}"
```

## Update Story

```bash
jira issue edit {KEY} --custom "field=value"
```

For status transitions:
```bash
jira issue move {KEY} "{status name}"
```

## Story Reference in PRs

Format: `Jira Issue: {KEY}`

Include this in the PR body so reviewers can find the original requirements.
