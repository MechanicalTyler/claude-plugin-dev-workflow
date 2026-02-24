---
name: debugger
description: "Debug workflow or address rework on a story. Use when /start debugger or /start rework is invoked."
---

# Debugger / Developer / Rework

**Role:** Unified skill for debugging, development, and rework — mode detected from arguments

## Arguments: $ARGUMENTS

## Mode Detection

Parse arguments:
- **Empty** → **Debugging mode** — wait for user to describe the bug
- **Story ID only** (e.g., `sc-12345`) → **Development mode** — implement the story
- **Story ID + `--rework`** (e.g., `sc-12345 --rework`) → **Rework mode** — address story comments

---

## CRITICAL: Mandatory Rules

### Reality Filter
- Never present generated, inferred, speculated, or deduced content as fact
- Label unverified content: [Inference] [Speculation] [Unverified]
- Ask for clarification if information is missing. Do not guess or fill gaps

### Development Standards
1. **Test Driven Development** — write failing tests first, then implement
2. **Respect existing architecture** — study codebase before making changes
3. **No placeholder code** — implement full functionality; if unable, ask for help
4. **For database changes** — update DAO, Entity classes, and Migrations
5. **Test before completion** — all tests must pass before pushing

### Commit and PR Process
- **NEVER** commit to main; **NEVER** skip commit hooks
- **NO boilerplate** — no AI attribution in commits, PRs, or comments
- **Commit frequently**, **always push** when done
- **Create PR** after implementation — clean, professional description

---

## Debugging Mode

*Triggered when: no arguments provided*

### Step 1: Gather Information
Wait for the user to describe the bug. Do not proactively ask questions — let them guide the session.

You need:
- What is broken
- Expected vs actual behavior
- Steps to reproduce
- Affected environment(s)

### Step 2: Reproduce the Bug
- Follow the reproduction steps provided
- Verify the issue occurs as described
- Document exact conditions that trigger it

### Step 3: Collect Evidence
- Gather logs, error messages, stack traces
- Check recent git history: `git log --oneline -20`
- Review test coverage for affected areas

### Step 4: Root Cause Analysis
- Trace code execution to where behavior diverges
- Use Read tool to examine relevant files
- Use Grep to find related patterns
- Document findings with file:line references

### Step 5: Fix with TDD
- **Write failing test first** that reproduces the bug
- Verify test fails before fixing
- Implement fix following existing patterns
- Verify test passes after fix
- Run full test suite to check for regressions

### Step 6: Commit and Create PR
- Branch naming: `fix/[brief-description]`
- PR body: describe the bug, root cause (with file:line refs), and fix
- Include test results as evidence

---

## Development Mode

*Triggered when: story ID provided (no --rework flag)*

### Step 1: Load Adapters and Story
1. Read `~/.claude/dev-workflow/config.json` for `pm_adapter` and `notes_adapter`
2. Load PM adapter: check `~/.claude/skills/pm-adapter/{pm_adapter}.md` first (user override); fall back to `skills/pm-adapter/{pm_adapter}.md` → fetch story by ID
3. Load notes adapter: check `~/.claude/skills/notes-adapter/{notes_adapter}.md` first (user override); fall back to `skills/notes-adapter/{notes_adapter}.md` → read Claude Instructions spec
4. **If spec not found:** STOP and ask user to run `/start writer {story-id}` first

### Step 2: Branch
- Check current branch — if on `main`, create a new branch: `feature/`, `fix/`, or `chore/`
- If already on a feature branch, check if a PR exists

### Step 3: Implement with TDD
- Write failing tests first based on acceptance criteria
- Implement following Claude Instructions step-by-step
- Run tests after each significant change
- Follow existing architecture patterns

### Step 4: Commit and Push
- Commit frequently with descriptive messages
- Run all tests and fix any failures before pushing
- Push to remote branch

### Step 5: Create PR
- Title: concise and descriptive
- Body: summary + story reference (PM adapter format) + testing steps from Claude Instructions
- NO AI-generated boilerplate

---

## Rework Mode

*Triggered when: story ID + `--rework` provided*

### Step 1: Load Context
1. Read `~/.claude/dev-workflow/config.json` for `pm_adapter` and `notes_adapter`
2. Load PM adapter: check `~/.claude/skills/pm-adapter/{pm_adapter}.md` first (user override); fall back to `skills/pm-adapter/{pm_adapter}.md` → fetch story by ID (title, description, acceptance criteria, **all comments**)
3. Load notes adapter: check `~/.claude/skills/notes-adapter/{notes_adapter}.md` first (user override); fall back to `skills/notes-adapter/{notes_adapter}.md` → read Claude Instructions spec (best-effort — continue even if missing; rework requirements from comments take precedence)

### Step 2: Extract Rework Requirements

Parse all story comments chronologically (oldest to newest). Identify:
- Reviewer change requests
- Rejection notes or reasons
- Bug reports or edge cases found
- Requests to update logic, naming, structure, or tests

**Summarize as a numbered checklist before writing any code:**

```
## Rework Items Identified

1. [Summary of item from comment dated YYYY-MM-DD by @author]
2. [Summary of another item]
```

**If no comments indicate rework:**
- STOP and ask: "I don't see any comments describing rework requirements. Can you clarify what needs to be changed?"

### Step 3: Create New Branch

**ALWAYS** create a new `fix/` branch — never reuse an existing rework branch.

```bash
git checkout -b fix/sc-XXXXX-rework
# Or more descriptive: fix/sc-XXXXX-address-review-feedback
```

### Step 4: Implement with TDD

- Address ONLY the items from story comments — no scope creep
- Write failing tests for each rework item, then fix
- Commit frequently with descriptive messages

### Step 5: Create PR

PR body format:
```markdown
## Summary
[What was reworked and why]

[Story reference in PM adapter format]

## Rework Items Addressed
1. [Item 1 from comments — what was changed]
2. [Item 2 from comments — what was changed]

## How to Test
- [ ] [Step to verify rework item 1]
- [ ] [Step to verify rework item 2]
- [ ] Existing tests still pass
```

---

## File Operations

- **Use Write tool for files** — never use `cat` or `echo` with redirection
- **Stay within repository** — do not `cd` outside the project
- **Never give up** when debugging. If stuck, ask for help.
