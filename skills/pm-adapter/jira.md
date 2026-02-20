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
- **Summary**: value on the `Summary:` labeled line
- **Description**: content under the `Description:` label
- **Issue type**: value on the `Type:` labeled line
- **Status**: value on the `Status:` labeled line
- **Acceptance Criteria**: look in a dedicated `Acceptance Criteria:` custom field, or as a structured section (heading or bullet list) within the description
- **Comments**: content under the `Comments:` section at the bottom

## Post Comment

```bash
jira issue comment add {KEY} --body "{text}"
```

## Update Story

Standard fields use named flags:
```bash
jira issue edit {KEY} --priority High
jira issue edit {KEY} --assignee username
```

Custom fields use `--custom`:
```bash
jira issue edit {KEY} --custom "story-points=5"
```

For status transitions:
```bash
jira issue move {KEY} "In Progress"
```

## Story Reference in PRs

Format: `Jira Issue: {KEY}`

Include this in the PR body so reviewers can find the original requirements.
