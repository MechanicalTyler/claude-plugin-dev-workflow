# PM Adapter: Shortcut

Story ID format: `sc-XXXXX` or numeric `XXXXX` (strip "sc-" prefix before MCP calls)

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
