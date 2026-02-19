# Notes Adapter: Local Filesystem

Stores specs in the repository under `docs/specs/`.

No external tools required — specs travel with the code.

## Spec path

```
{repo_root}/docs/specs/{story-id}.md
```

Example: for story sc-12345:
```
/path/to/your/repo/docs/specs/sc-12345.md
```

Get repo root with:
```bash
git rev-parse --show-toplevel
```

## Read spec

Use the Read tool with the full spec path. If the file does not exist, return "not found".

## Write spec

1. Create the specs directory if it doesn't exist:
   ```bash
   mkdir -p {repo_root}/docs/specs
   ```
2. Use the Write tool to write the spec content to the full path.

## Note

Unlike the Obsidian adapter, local specs are per-story (not per-service). If a story spans multiple services, store the full spec in one repo and reference it from others.
