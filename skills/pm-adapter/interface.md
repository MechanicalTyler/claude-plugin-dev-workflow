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
2. Load adapter with user-override lookup:
   - First: Read `~/.claude/skills/pm-adapter/{pm_adapter}.md` — if found, use it (user override)
   - Fallback: load `skills/pm-adapter/{pm_adapter}.md` (plugin built-in)
   - If neither exists: STOP — "PM adapter '{name}' not found. Check ~/.claude/dev-workflow/config.json and ensure ~/.claude/skills/pm-adapter/{name}.md exists or the name matches a built-in adapter."
3. Follow adapter's instructions for all PM operations

## User Adapters

Place a file at `~/.claude/skills/pm-adapter/{name}.md` to create a custom adapter. Set `pm_adapter: "{name}"` in config.

User adapters take precedence over plugin adapters with the same name. This allows you to override any built-in adapter or create one for an unsupported PM tool.

Your adapter must implement the same interface: Fetch story, Post comment, Update story, Story reference format.
