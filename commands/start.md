# Start — Role Dispatcher

Initialize specialized agent roles for development workflow.

## Arguments: $ARGUMENTS

Parse first argument as role, remaining as role-specific parameters.

---

## Dispatch Rules

**`developer [story-id]`** → Invoke dev-workflow:developer subagent (pass story ID if provided)

**`writer story-id`** → Invoke dev-workflow:writer subagent with story ID

**`reviewer {PR}`** → Invoke dev-workflow:reviewer subagent with PR number

**`tester {PR}`** → Invoke dev-workflow:tester subagent with PR number

**`debugger`** → Invoke dev-workflow:debugger subagent (empty args = debugging mode)

**`rework story-id`** → Invoke dev-workflow:debugger subagent with story ID + `--rework` flag

**No role or invalid role** → Display usage below

---

## Usage

```
/start developer              # Development workflow (no story, just rules)
/start developer sc-12345     # Development workflow with story context
/start writer sc-12345        # Transform story into Claude Instructions spec
/start reviewer 42            # Review PR #42 against story requirements
/start tester 42              # Test PR #42 in dev environment
/start debugger               # Debug workflow (prompts for problem description)
/start rework sc-12345        # Address rework on story sc-12345 (reads story comments)
```

---

## Notes

- **writer** requires a story ID — no default
- **reviewer** and **tester** require a PR number
- **developer** without a story ID runs the workflow with general rules only (no PM/notes adapter loading)
- **rework** always creates a new `fix/` branch — never reuses existing branches
- All roles read `~/.claude/dev-workflow/config.json` to determine PM and notes adapters
