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
