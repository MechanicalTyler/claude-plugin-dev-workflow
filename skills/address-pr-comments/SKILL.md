---
name: address-pr-comments
description: "Address PR review feedback in the current session — reads new comments, file-level inline comments, and review decisions since the last commit, then implements the required changes and replies to the PR with a summary of fixes. Use when a user asks to address PR comments, respond to review feedback, fix review notes, or iterate on a PR."
---

# Address PR Comments

**Role:** Address review feedback on the current PR — read, implement, and respond

Read `skills/shared/standards.md` — these mandatory rules govern this entire session.

Compact the conversation before continuing — you are about to iterate on existing work.

---

## Step 1: Identify the PR

If no PR number is provided in context:

```bash
gh pr status --json currentBranch
```

Use the current branch's open PR. If no PR is open, ask the user for the PR number.

---

## Step 2: Load New Feedback

Fetch all comment types since the last commit. Use the actual PR number in every command.

**Conversation comments:**
```bash
gh api repos/{owner}/{repo}/issues/{PR_NUMBER}/comments
```

**Inline file comments:**
```bash
gh api repos/{owner}/{repo}/pulls/{PR_NUMBER}/comments
```

**Reviews (state + body):**
```bash
gh api repos/{owner}/{repo}/pulls/{PR_NUMBER}/reviews
```

Filter to comments created or updated after the last commit timestamp:
```bash
git log -1 --format=%cI
```

Deduplicate: if an inline comment is also referenced in a review body, address it once.

---

## Step 3: Summarize Required Changes

Before writing any code, list every actionable item as a numbered checklist:

```
## Changes Required

1. [Author @foo, file.ts:42] — [what they asked for]
2. [Author @bar, review body] — [what they asked for]
```

If no actionable feedback is found, inform the user and stop.

---

## Step 4: Implement Changes

Address each item from the checklist:

- Apply the full RED-GREEN-REFACTOR cycle for code changes:
  > Invoke Skill: `superpowers:test-driven-development`
- Address items in checklist order — do not skip or defer
- Do not change anything outside the scope of reviewer feedback
- Commit frequently with descriptive messages referencing the item being addressed

---

## Step 5: Verify

> Invoke Skill: `superpowers:verification-before-completion`
>
> Verify with fresh execution:
> - All tests pass (run the full suite now)
> - Each checklist item from Step 3 has been addressed
> - Code is pushed to remote

---

## Step 6: Reply to PR

Post a comment on the PR summarizing what was done. Call out anywhere you deviated from what the reviewer requested and explain why.

Comment format:
```markdown
## Changes Addressed

1. [Item 1] — [what was changed and where: file.ts:42]
2. [Item 2] — [what was changed and where]

## Deviations
[Omit section entirely if none. Otherwise explain each deviation and reasoning.]
```

```bash
gh pr comment {PR_NUMBER} --body "..."
```
