# PM Adapter: Shortcut

Story ID format: `sc-XXXXX` or numeric `XXXXX` (strip "sc-" prefix before MCP calls). Prefer `sc-XXXXX` any time `XXXXX` is not explicitly required.

## Fetch Story

MCP tool: `mcp__shortcut__stories-get-by-id` with numeric story ID

Returns: name, description, comments[], workflow_state, story_type

Also fetch comments with: `mcp__shortcut__stories-get-history` for full comment thread

## Post Comment

MCP tool: `mcp__shortcut__stories-create-comment` with story_id (numeric) and text

## Update Story

MCP tool: `mcp__shortcut__stories-update` with story_id (numeric) and fields object

## Story Reference in PRs

Format: `Shortcut Story: sc-XXXXX`

Include this in the PR body so reviewers can find the original requirements.

## Story reference in notes Adapter

Format: `sc-XXXXX`

## Create Story

MCP tool: `mcp__shortcut__stories-create` with fields:
- `name`: story title
- `description`: full markdown body (see body format below)
- `story_type`: infer from context ("feature" for new capabilities, "bug" for fixes, "chore" for maintenance)
- `workflow_state_id`: use default workflow — call `mcp__shortcut__workflows-get-default` to get the starting state ID

### Story body format

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

Return: the created story's public ID (e.g., `sc-601`) and `app_url` for confirmation.
