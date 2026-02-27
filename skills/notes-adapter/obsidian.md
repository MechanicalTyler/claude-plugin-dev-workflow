# Notes Adapter: Obsidian

Configuration keys (from `~/.claude/dev-workflow/config.json` → `adapters.obsidian`):
- `vault_path`: absolute path to vault (e.g., `/Users/you/Documents/Obsidian/MyVault`)
- `prompts_dir`: relative path within vault (default: `Engineering/Prompts`)

## Spec path

```
{vault_path}/{prompts_dir}/{story-id}/{service-name}.md
```

Example: for story 12345 in service `api-server`:
```
/Users/you/Documents/Obsidian/MyVault/Engineering/Prompts/12345/api-server.md
```

## Service name detection

Get the service name from the current repo:
```bash
git rev-parse --show-toplevel | xargs basename
```

## Read spec

Follow these steps in order:

1. Run the service name detection command above to get `{service-name}`
2. Construct the full path: `{vault_path}/{prompts_dir}/{story-id}/{service-name}.md`
3. Use the Read tool to read the file at that path
4. If not found, use Glob to search `{vault_path}/{prompts_dir}/{story-id}/*.md` as a fallback:
   - If exactly one file is found, read it
   - If multiple files are found, ask the user which service to use
   - If no files are found, return "not found"

## Write spec

1. Create the parent directory if it doesn't exist:
   ```bash
   mkdir -p {vault_path}/{prompts_dir}/{story-id}
   ```
2. Use the Write tool to write the spec content to the full path.

## Multi-service support

The same story can have different specs for different services. Each lives in its own file:
```
12345/
  api-server.md
  web-frontend.md
  worker.md
```
