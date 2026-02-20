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
