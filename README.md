# dev-workflow

Role-based development workflow subagents with pluggable PM and notes adapters.

## Roles

| Command | Role | Purpose |
|---------|------|---------|
| `/start developer [story-id]` | Developer | Branch, implement with TDD, commit, create PR |
| `/start writer story-id` | Writer | Fetch story → analyze codebase → write Claude Instructions spec |
| `/start reviewer PR` | Reviewer | Multi-perspective PR review against story requirements |
| `/start tester PR` | Tester | Functional testing with evidence gathering |
| `/start debugger` | Debugger | Debug-first workflow (describe bug → investigate → TDD fix) |
| `/start rework story-id` | Rework | Read story comments as rework items → fix → new PR |

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
