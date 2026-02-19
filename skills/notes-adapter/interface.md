# Notes Adapter Interface

A notes adapter tells you where to read/write implementation specs for stories.

## Configuration

Read `~/.claude/dev-workflow/config.json`. The `notes_adapter` field names which adapter to use.

## Required Capabilities

**1. Read spec** — given story ID and service name, return spec content (or "not found")

**2. Write spec** — given story ID, service name, and content, persist the spec

## Service name detection

```bash
git rev-parse --show-toplevel | xargs basename
```

Run this to determine the current service name. Use it as part of the spec path.

## How to use

1. Read config to get `notes_adapter` value
2. Load that adapter's skill (e.g., obsidian.md, local.md)
3. Follow adapter's instructions for all read/write spec operations
