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

**Native attachment:** Jira detects the issue key automatically from (requires the GitHub for Jira app installed):
- Branch names containing `PROJ-###` anywhere (e.g., `feature/DEV-123-fix-login`, `DEV-123-fix-login`)
- PR title containing `PROJ-###`
- Commit messages containing `PROJ-###`

Any one of the above is sufficient to show the PR in the Jira issue's Development panel. No special delimiters or brackets needed — `PROJ-###` as a substring is detected.

**Recommended:** Include the issue key in the branch name to trigger the native link automatically.

**Fallback reference in PR body** (for reviewers without Jira access): `Jira Issue: {KEY}`

## Finding PRs linked to a story

The Jira dev-status API returns linked PRs (requires Jira API token):

```bash
# Step 1: get the numeric issue ID
ISSUE_ID=$(curl -su "user@example.com:$JIRA_API_TOKEN" \
  "https://your-domain.atlassian.net/rest/api/3/issue/{KEY}" \
  | jq -r '.id')

# Step 2: fetch linked pull requests
curl -su "user@example.com:$JIRA_API_TOKEN" \
  "https://your-domain.atlassian.net/rest/dev-status/latest/issue/detail?issueId=$ISSUE_ID&applicationType=GitHub&dataType=pullrequest" \
  | jq '.detail[].pullRequests[]'
```

Or as a fallback, search GitHub:
```bash
gh pr list --state all --search "{KEY}"
```

## Story reference in notes Adapter

Format: `{KEY}` — use the full issue key as-is (e.g., `PROJ-123`, `ENG-456`)

## Create Story

Use the `jira` CLI (Ankitpokhrel) to create a new issue. Write the body to a temp file first to avoid shell interpolation issues with multi-line markdown:

```bash
# Write body to .scratch/tmp/jira-issue-body.md using the Write tool first, then:
jira issue create --project {PROJECT_KEY} --summary "{title}" --body-file .scratch/tmp/jira-issue-body.md --type Story
```

**Note:** `PROJECT_KEY` is the Jira project prefix (e.g., `ENG`, `PROJ`). If not available in context, ask the user for the project key before creating the story.

### Body format

Construct the body as:

```
## Original Request
{originalRequest}

---

## Story

{description}

**Repo to modify:** {repoToModify}

**Repos to reference:** {reposToReference joined with ", " or "(none)" if empty}

**Acceptance Criteria**
- [ ] {ac item 1}
- [ ] {ac item 2}
...

**Testing Instructions**
1. {step 1}
2. {step 2}
...
```

Return: the created issue key (e.g., `PROJ-123`) and URL for confirmation.
