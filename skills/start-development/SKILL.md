---
name: dev-workflow:start-development
description: "Full-stack development workflow with story context loading, TDD, automated planning, subagent execution, and PR creation. Use this skill whenever implementing features, fixing bugs, or any hands-on coding task. Always use this when a user says 'implement', 'build', 'code up', 'add feature', 'start dev', or provides a story ID to work from. Works with or without a PM story."
---

# Start Development

**Role:** Developer — implement features, fix bugs, and maintain code quality

Read `skills/shared/standards.md` — these mandatory rules govern this entire session.

Read `skills/shared/adapter-loading.md` — adapter loading procedures referenced in PM Context.

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
2. **Brainstorm requirements** — Before writing any code, invoke brainstorming to clarify scope boundaries and identify risks within the stated requirements:
   > Invoke Skill: `superpowers:brainstorming`
   >
   > OVERRIDE: After brainstorming completes, do NOT invoke `superpowers:writing-plans` yet.
   > Return here and proceed to step 3.
3. **Plan implementation** — Invoke the planning skill:
   > Invoke Skill: `superpowers:writing-plans`
   >
   > OVERRIDE: The plan is a human-readable artifact saved to a local file — render it as a
   > standalone HTML document per the Output Format rules in `skills/shared/standards.md`, and
   > save to `./.scratch/tmp/YYYY-MM-DD-plan.html`.
   > NEVER save to `docs/` or any subdirectory (including `docs/superpowers/plans/`).
   > `.scratch/` is gitignored — this file must never be committed.
   > Use the brainstorming output and user's description as the feature description input.
4. **Implement** — Use the Development Standards below. Apply TDD for each distinct behavior.
5. **Commit and PR** — Follow the Commit and PR Process below. Include a clear description of what was built and why.

After planning, skip the "PM Context" and "Implementation Planning" sections — continue from **Development Standards**.

---

## PM Context (if story ID provided)

If you have a story ID:

1. Read `~/.claude/dev-workflow/config.json` to determine `pm_adapter` and `notes_adapter`
2. Load PM adapter per procedure in `skills/shared/adapter-loading.md` → fetch story via PM adapter instructions
3. Load notes adapter per procedure in `skills/shared/adapter-loading.md` → read Claude Instructions spec
4. **If spec not found:** STOP and ask user to invoke the Writer skill (`dev-workflow:write-spec`) with this story ID first
5. Use spec as the primary implementation guide

---

## Implementation Planning (when story ID and spec are loaded)

After loading the Claude Instructions spec, invoke the planning skill:

> Invoke Skill: `superpowers:writing-plans`
>
> OVERRIDE: The plan is a human-readable artifact saved to a local file — render it as a
> standalone HTML document per the Output Format rules in `skills/shared/standards.md`, and
> save to `./.scratch/tmp/YYYY-MM-DD-<story-id>-plan.html`.
> NEVER save to `docs/` or any subdirectory (including `docs/superpowers/plans/`).
> `.scratch/` is gitignored — this file must never be committed.
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

Before declaring work complete, run the steps below in order.

### Terraform Plan Check (if applicable)

Detect whether terraform files were changed in this branch. First, fetch the remote to ensure the default branch ref is up to date:

```bash
git fetch origin
```

Then diff against the remote default branch:

```bash
git diff origin/HEAD --name-only
```

Look for any files ending in `.tf` or located inside a `tf/` path.

**If no terraform files changed:** Skip this section and proceed to the verification skill below.

**If terraform files changed:** Check whether a CI terraform plan workflow ran for this branch. First get the current branch name:

```bash
git branch --show-current
```

Then check CI runs:

```bash
gh-as-app.sh developer run list --branch <current-branch> --json conclusion,status,name,createdAt,workflowName --limit 25
```

Look for any workflow whose name contains "terraform" (case-insensitive).

- **CI terraform run found and passed:** No additional action needed — CI has already validated the plan. Continue to the verification skill below.
- **CI terraform run found and failed:** The CI terraform plan failed. Do not declare work complete — fix the plan failure and re-run CI before proceeding.
- **CI terraform workflow is still `in_progress`:** Wait for it to complete. Re-run the `gh run list` command every 2 minutes until the run reaches a terminal conclusion (success, failure, or cancelled). Cap the wait at 30 minutes total. If the run has not completed after 30 minutes, note a warning and continue to the verification skill below.
- **No CI terraform run found:** Run `terraform plan` directly. Use the same directory detection as review-pr Phase 3: check for `tf/` first, then `terraform/`, then fall back to the directory of the changed `.tf` files. For example, if `tf/` exists:

  ```bash
  terraform -chdir=tf/ plan
  ```

  - **`terraform` is not installed on the machine:** Note that terraform validation was not possible due to the missing CLI. Continue to the verification skill below with a warning.
  - **Plan exits with a non-zero exit code:** Do not declare work complete — fix the plan failure before proceeding.
  - **Plan exits with exit code 0:**

    Capture the full plan output. Before including it in the PR description, consider omitting any sensitive attribute values (ARNs, account IDs, IP ranges, secret resource references) from the output.

    If a PR already exists for this branch, append the plan output to the PR description using `--body-file` to avoid shell injection from plan output that may contain backticks or `$()`:

    1. Read the current PR body:

       ```bash
       gh-as-app.sh developer pr view --json body -q .body
       ```

    2. Write the combined body (existing content + the Terraform Plan section) to `.scratch/pr-body-updated.txt` using the Write tool.

    3. Update the PR description:

       ```bash
       gh-as-app.sh developer pr edit --body-file .scratch/pr-body-updated.txt
       ```

    4. Delete the scratch file:

       ```bash
       rm .scratch/pr-body-updated.txt
       ```

    If no PR exists yet, save the plan output to `.scratch/terraform-plan.txt` — you will include it in the PR description when you create the PR.

  Then continue to the verification skill below.

### Final Verification

> Invoke Skill: `superpowers:verification-before-completion`
>
> Verify with fresh command execution (not memory of previous runs):
> - All tests pass (run the full test suite now)
> - Code is pushed to remote
> - PR exists and is not draft

---

## Adversarial Review (when story ID provided)

If the "No Story ID Path" was used (no PM story), skip this section — there is no independent story/spec to form expectations against.

Read and follow the adversarial review procedure in `skills/shared/adversarial-review.md` with these context variables:

- `story_id`: the story ID from PM Context
- `review_target`: "code changes on branch"
- `review_context`: the Claude Instructions spec loaded during PM Context

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
