---
name: developer
description: "Full-stack development workflow with story context loading, TDD, automated planning, subagent execution, and PR creation. Use this skill whenever implementing features, fixing bugs, or any hands-on coding task. Always use this when a user says 'implement', 'build', 'code up', 'add feature', 'start dev', or provides a story ID to work from. Works with or without a PM story."
---

# Developer

**Role:** Developer — implement features, fix bugs, and maintain code quality

Read `skills/shared/standards.md` — these mandatory rules govern this entire session.

Read the CLAUDE.md file in this repository before starting.

---

## Branch Management

- **ALWAYS** start by checking out a new branch with prefix: `feature/`, `fix/`, or `chore/`
- Branch names should be descriptive: `feature/slack-monitoring`, `fix/docker-permissions`, `chore/update-dependencies`
- Check which branch you're on first. If not on `main`, you may already be on the correct branch — check if a PR is already open.

---

## No Story ID Path

If **no story ID** was provided, work directly with the user to define and scope the task:

1. **Understand the task** — If the request is vague or missing, use `AskUserQuestion` to clarify what needs to be built or changed.
2. **Brainstorm requirements** — Before writing any code, invoke brainstorming to surface edge cases, scope, and approach:
   > Invoke Skill: `superpowers:brainstorming`
   >
   > OVERRIDE: After brainstorming completes, do NOT invoke `superpowers:writing-plans` yet.
   > Return here and proceed to step 3.
3. **Plan implementation** — Invoke the planning skill:
   > Invoke Skill: `superpowers:writing-plans`
   >
   > OVERRIDE: Save plan to `./.scratch/tmp/YYYY-MM-DD-plan.md`.
   > Use the brainstorming output and user's description as the feature description input.
4. **Implement** — Use the Development Standards below. Apply TDD for each distinct behavior.
5. **Commit and PR** — Follow the Commit and PR Process below. Include a clear description of what was built and why.

After planning, skip the "PM Context" and "Implementation Planning" sections — continue from **Development Standards**.

---

## PM Context (if story ID provided)

If you have a story ID:

1. Read `~/.claude/dev-workflow/config.json` to determine `pm_adapter` and `notes_adapter`
2. Load PM adapter: check `~/.claude/skills/pm-adapter/{pm_adapter}.md` first (user override); fall back to `skills/pm-adapter/{pm_adapter}.md` → fetch story via PM adapter instructions
3. Load notes adapter: check `~/.claude/skills/notes-adapter/{notes_adapter}.md` first (user override); fall back to `skills/notes-adapter/{notes_adapter}.md` → read Claude Instructions spec
4. **If spec not found:** STOP and ask user to invoke the Writer skill (`dev-workflow:writer`) with this story ID first
5. Use spec as the primary implementation guide

---

## Implementation Planning (when story ID and spec are loaded)

After loading the Claude Instructions spec, invoke the planning skill:

> Invoke Skill: `superpowers:writing-plans`
>
> OVERRIDE: Save plan to `./.scratch/tmp/YYYY-MM-DD-<story-id>-plan.md` (not `docs/plans/`).
> Use the Claude Instructions spec as the feature description input.

Then invoke subagent-driven execution:

> Invoke Skill: `superpowers:subagent-driven-development`
>
> IMPORTANT OVERRIDE: Proceed automatically with subagents without asking the user for
> confirmation. Dispatch subagents and proceed.
>
> IMPORTANT OVERRIDE: Do NOT invoke `superpowers:using-git-worktrees`. Develop in the
> current branch. Pass this override to any nested `finishing-a-development-branch`
> invocation.

---

## Development Standards

1. **Test Driven Development** — Write failing tests first. Tests should fail until implementation is correct, then pass.

   Apply the full RED-GREEN-REFACTOR cycle:
   > Invoke Skill: `superpowers:test-driven-development`
   > Use this for each distinct behavior being implemented.
2. **Respect existing architecture patterns** — Study the codebase structure before making changes
3. **No placeholder code** — Always implement full functionality. If unable, stop and ask for help
4. **For database changes** — Update appropriate DAO, Entity classes, and Migrations
5. **Test before completion** — Run tests and fix all errors before considering work done

---

## Debugging and Problem Solving

- Never give up when debugging. If stuck, ask for help
- If unable to access a screenshot, mockup, or attachment referenced in requirements — STOP and ask the user. Do not proceed with incomplete data.
- Use `gh api` instead of `gh pr` when reading PR comments and file comments
- Always run `git status` after committing to ensure nothing was missed

---

## Commit and PR Process

- **Commit frequently** with descriptive messages explaining what was accomplished
- **NEVER** commit to main
- **NEVER** skip commit hooks
- **NO boilerplate** — Never include "Co-Authored by Claude", "Generated with Claude Code", or any AI attribution in commits or PRs
- **ALWAYS commit and push** after completing work — never leave work uncommitted
- **MANDATORY: Create PR after successful implementation** using `gh pr create`
- **Clean PR descriptions** — focus on what was changed and why

---

## PR Creation Requirements

When creating the PR:
- Title should be concise and descriptive
- Body must include:
  - **Summary**: Brief description of changes
  - **Story Reference**: Link using PM adapter's "Story Reference in PRs" format (omit this section if there is no story ID)
  - **How to Test**: Testing steps from Claude Instructions if available, otherwise based on changes made
- NO AI-generated boilerplate or mentions of AI tools

---

## Internal Code Review (when story ID provided)

After the subagent-driven implementation completes, invoke a code review before creating the PR:

> Invoke Skill: `superpowers:requesting-code-review`
>
> Provide the code-reviewer subagent with:
> - The Claude Instructions spec as the expected-functionality reference
> - The story acceptance criteria
> - The diff of all changes made during implementation
>
> Address any required changes before proceeding to PR creation.

Note: `superpowers:subagent-driven-development` includes per-task spec and quality reviews
internally. This step adds a final whole-implementation review before the PR is opened.

---

## Pre-Completion Verification

Before declaring work complete, run the verification skill:

> Invoke Skill: `superpowers:verification-before-completion`
>
> Verify with fresh command execution (not memory of previous runs):
> - All tests pass (run the full test suite now)
> - Code is pushed to remote
> - PR exists and is not draft

---

## Completion Criteria

- All changes are committed with clean messages
- All tests pass
- Code is pushed to remote branch
- PR is created with clean, professional description linking the story

---

## Debugging and Problem Solving

- Never give up when debugging. If stuck, ask for help
- If unable to access a screenshot, mockup, or attachment referenced in requirements — STOP and ask the user. Do not proceed with incomplete data.
- Use `gh api` instead of `gh pr` when reading PR comments and file comments
- Always run `git status` after committing to ensure nothing was missed
