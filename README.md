# dev-workflow

Role-based development workflow subagents with pluggable PM and notes adapters.

## Prerequisites

This plugin requires the [superpowers plugin](https://github.com/obra/superpowers) to be installed:

```
/plugin install superpowers@superpowers-marketplace
```

The superpowers plugin provides core methodology skills (TDD, systematic debugging, brainstorming, verification gates, subagent orchestration) that are invoked throughout the dev-workflow skill phases.

## Roles

| Command | Skill | Purpose |
|---------|-------|---------|
| `/start start-development [story-id]` | start-development | Branch, implement with TDD, commit, create PR |
| `/start write-spec story-id` | write-spec | Fetch story → analyze codebase → write Claude Instructions spec |
| `/start review-pr PR` | review-pr | Multi-perspective PR review against story requirements |
| `/start test-pr PR` | test-pr | Functional testing with evidence gathering |
| `/start start-debugging` | start-debugging | Debug-first workflow (describe bug → investigate → TDD fix) |
| `/start start-debugging story-id --rework` | start-debugging | Read story comments as rework items → fix → new PR |
| `/start create-story` | create-story | Interview user → draft story → submit to PM tool |

## Configuration

Create `~/.claude/dev-workflow/config.json`:

```json
{
  "pm_adapter": "shortcut",
  "notes_adapter": "obsidian",
  "adapters": {
    "obsidian": {
      "vault_path": "/path/to/your/vault",
      "prompts_dir": "Engineering/Prompts"
    },
    "shortcut": {
      "story_id_prefix": "sc-"
    }
  },
  "deploy_command": "Run the dev CI workflow in GitHub Actions"
}
```

## Adapters

**PM adapters** (`skills/pm-adapter/`): `shortcut`, `linear`, `github-issues`

**Notes adapters** (`skills/notes-adapter/`): `obsidian`, `local`

## Custom Adapters

You can override any built-in adapter or create a new one by placing a file in `~/.claude/skills/`:

- PM adapters: `~/.claude/skills/pm-adapter/{name}.md`
- Notes adapters: `~/.claude/skills/notes-adapter/{name}.md`

Set the matching name in your config:

```json
{
  "pm_adapter": "my-pm-tool",
  "notes_adapter": "my-notes-tool"
}
```

User adapters in `~/.claude/skills/` take precedence over plugin adapters with the same name. This means you can override a built-in adapter (e.g., create `~/.claude/skills/pm-adapter/shortcut.md` to customize Shortcut behavior) or add support for a new tool entirely.

Your adapter must implement the same interface as built-in adapters — see `skills/pm-adapter/interface.md` or `skills/notes-adapter/interface.md` for the required capabilities.

## Installation

```json
{
  "enabledPlugins": {
    "dev-workflow@local": { "path": "/path/to/dev-workflow" }
  }
}
```

## License

MIT
