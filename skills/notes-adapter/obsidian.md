# Notes Adapter: Obsidian

Configuration keys (from `~/.claude/dev-workflow/config.json` → `adapters.obsidian`):
- `vault_path`: absolute path to vault (e.g., `/Users/you/Documents/Obsidian/MyVault`)
- `prompts_dir`: relative path within vault (default: `Engineering/Prompts`)

## Spec path

```
{vault_path}/{prompts_dir}/sc-{story-id}/{service-name}.md
```

Example: for story sc-12345 in service `api-server`:
```
/Users/you/Documents/Obsidian/MyVault/Engineering/Prompts/sc-12345/api-server.md
```

## Read spec

Use the Read tool with the full spec path. If the file does not exist, return "not found".

## Write spec

1. Create the parent directory if it doesn't exist:
   ```bash
   mkdir -p {vault_path}/{prompts_dir}/sc-{story-id}
   ```
2. Use the Write tool to write the spec content to the full path.

## Multi-service support

The same story can have different specs for different services. Each lives in its own file:
```
sc-12345/
  api-server.md
  web-frontend.md
  worker.md
```
