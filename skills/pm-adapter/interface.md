# PM Adapter Interface

A PM adapter tells you how to interact with the project management tool for this project.

## Configuration

Read `~/.claude/dev-workflow/config.json`. The `pm_adapter` field names which adapter to use.

## Required Capabilities

**1. Fetch story** — given a story ID, return: title, description, acceptance criteria, story type, comments

**2. Post comment** — given story ID and text, add a comment to the story

**3. Update story** — given story ID and field/value pairs, update the story

**4. Story ID in PRs** — each adapter specifies how to reference the story in PR descriptions

## How to use

1. Read config to get `pm_adapter` value
2. Load that adapter's skill (e.g., shortcut.md, linear.md, github-issues.md)
3. Follow adapter's instructions for all PM operations
