# PM Adapter: Linear

Story ID format: `LIN-XXX` or team-prefixed `TEAM-XXX`

## Fetch Story

Use Linear MCP if available, otherwise use Linear GraphQL API with your API token.

Key fields to retrieve: title, description, comments, state, type, assignee

GraphQL query example:
```graphql
query Issue($id: String!) {
  issue(id: $id) {
    title
    description
    state { name }
    comments { nodes { body createdAt } }
  }
}
```

## Post Comment

Linear MCP comment tool, or GraphQL mutation:
```graphql
mutation CreateComment($issueId: String!, $body: String!) {
  commentCreate(input: { issueId: $issueId, body: $body }) {
    success
  }
}
```

## Update Story

Linear MCP update tool, or GraphQL mutation for state/label changes.

## Story Reference in PRs

Format: `Linear Issue: LIN-XXX`

Include this in the PR body so reviewers can find the original requirements.

## Story reference in notes Adapter

Format: `LIN-XXX` (or team-prefixed `TEAM-XXX` — use the same format that was passed in)

## Create Story

Use the Linear MCP create tool if available, otherwise use the GraphQL `issueCreate` mutation:

```graphql
mutation CreateIssue($teamId: String!, $title: String!, $description: String!) {
  issueCreate(input: {
    teamId: $teamId
    title: $title
    description: $description
  }) {
    success
    issue {
      identifier
      url
    }
  }
}
```

**Note:** `teamId` is required by Linear. If not available in context, ask the user for their Linear team ID before creating the story.

### Description format

Construct the description as:

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

Return: the created issue identifier (e.g., `LIN-42`) and URL for confirmation.
