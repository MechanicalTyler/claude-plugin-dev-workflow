# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**dev-workflow** is a Claude plugin (v2.0.0) that provides action-based development workflow orchestration with pluggable PM and notes adapters. It enables specialized workflows (Start Development, Story to Spec, Review PR, Test PR, Start Debugging, Create Story) through structured, quality-gated stages.

**Dependency:** Requires the `superpowers` plugin to be installed — it provides core methodology skills (TDD, debugging, brainstorming, subagent orchestration, verification).

## Architecture

### Role Dispatcher Pattern

Skills are invoked directly by name:

| Command | Skill | Purpose |
|---------|-------|---------|
| `/start start-development [story-id]` | `dev-workflow:start-development` | Feature implementation with TDD |
| `/start write-spec story-id` | `dev-workflow:write-spec` | Story → Claude Instructions spec |
| `/start review-pr PR-number` | `dev-workflow:review-pr` | Multi-perspective PR review |
| `/start test-pr PR-number` | `dev-workflow:test-pr` | Functional testing with evidence |
| `/start start-debugging` | `dev-workflow:start-debugging` | Bug investigation |
| `/start start-debugging story-id --rework` | `dev-workflow:start-debugging` (rework mode) | Address review feedback |
| `/start create-story` | `dev-workflow:create-story` | Interview user → draft → submit story |

### Adapter System

PM and notes integrations are **pluggable adapters** with a common interface defined in `skills/pm-adapter/interface.md` and `skills/notes-adapter/interface.md`.

**PM Adapters** (`skills/pm-adapter/`): Shortcut, Linear, Jira, GitHub Issues
**Notes Adapters** (`skills/notes-adapter/`): Local filesystem (`docs/specs/`), Obsidian vault

**Override mechanism:** User-provided adapters at `~/.claude/skills/pm-adapter/{name}.md` or `~/.claude/skills/notes-adapter/{name}.md` take precedence over built-in adapters.

### Superpowers Integration

Skills invoke superpowers throughout their workflows:
- `superpowers:writing-plans` — Implementation task structuring
- `superpowers:test-driven-development` — RED-GREEN-REFACTOR cycles
- `superpowers:brainstorming` — Requirement discovery and edge cases
- `superpowers:systematic-debugging` — Root cause analysis
- `superpowers:subagent-driven-development` — Parallel task execution
- `superpowers:requesting-code-review` / `superpowers:receiving-code-review` — Review workflows
- `superpowers:verification-before-completion` — 5-phase verification gate

## Key Design Decisions

- **review-pr skill has two modes:** First Review (exhaustive 4-perspective analysis) vs. Re-Review (verify previous `CHANGES_REQUESTED` items were addressed, new findings only if they meet the Critical Exception Threshold)
- **start-debugging skill is a unified 3-mode skill:** Debug mode (no args), Development mode (story-id), Rework mode (story-id + `--rework`)
- **Reality Filter:** All skills enforce labeling unverified content as `[Inference]`, `[Speculation]`, or `[Unverified]`
- **Config location:** User configuration lives at `~/.claude/dev-workflow/config.json`, not in the repo

## File Layout

```
skills/
  start-development/  # Full development workflow (TDD, subagents, PR)
  write-spec/      # Story → Claude Instructions spec transformation
  review-pr/          # Multi-perspective PR review with mode detection
  test-pr/            # Evidence-based functional testing
  start-debugging/    # Debug/dev/rework unified skill
  create-story/       # Interactive interview → PM story creation
  address-pr-comments/ # Address review feedback in current session
  pm-adapter/         # PM tool adapters + interface spec
  notes-adapter/      # Notes storage adapters + interface spec
.claude-plugin/       # Plugin manifest (plugin.json)
```

## Working on This Codebase

All content is Markdown skill definitions — there is no compiled code, no tests to run, and no build step. Changes are made by editing `.md` files in `skills/` and `commands/`.

When modifying a skill:
- Update the version in `.claude-plugin/plugin.json` if changing behavior
- Maintain phase numbering consistency within skills (phases are referenced by number in other skills and documentation)
- Preserve the adapter interface contracts in `interface.md` files — adapters must implement all required operations
- Test skill changes by invoking them with `/start <role>` in a target repository
