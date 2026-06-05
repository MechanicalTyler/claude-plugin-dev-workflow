# Notes Adapter: Local Filesystem

Stores specs in the repository under `docs/specs/`.

No external tools required — specs travel with the code.

## Spec path

Specs are standalone HTML documents (see Output Format in `skills/shared/standards.md`), so they use the `.html` extension.

```
{repo_root}/docs/specs/{story-id}.html
```

Example: for story sc-12345:
```
/path/to/your/repo/docs/specs/sc-12345.html
```

Get repo root with:
```bash
git rev-parse --show-toplevel
```

`repo_root` must resolve to the **specific repo being specced** — i.e. run `git rev-parse --show-toplevel` from inside that repo's directory, not from the folder the workflow was invoked from. When writing specs for multiple repos, resolve each repo's own root independently.

## Read spec

Use the Read tool with the full spec path, where `repo_root` is the root of the specific repo being read. If the file does not exist, return "not found".

## Write spec

1. Resolve `repo_root` for the specific repo being written (run `git rev-parse --show-toplevel` from inside that repo).
2. Create the specs directory if it doesn't exist:
   ```bash
   mkdir -p {repo_root}/docs/specs
   ```
3. Use the Write tool to write the spec content to the full path.

## Note

For stories that span multiple repos, write one spec into each repo's own `docs/specs/` directory. Every repo gets its own complete copy of the spec — nothing is shared or referenced across repos. Single-repo stories are unaffected: one spec at `{repo_root}/docs/specs/{story-id}.html`.
