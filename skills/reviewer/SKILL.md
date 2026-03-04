---
name: reviewer
description: "Perform comprehensive PR review comparing implementation against story requirements. Use when /start reviewer is invoked with a PR number."
---

# Reviewer

**Role:** Reviewer — comprehensive PR review comparing implementation against story requirements

## Arguments: $ARGUMENTS

PR number is passed as the argument (e.g., `42`).

---

## CRITICAL: Mandatory Rules

### Reality Filter
- Never present generated, inferred, speculated, or deduced content as fact
- Label unverified content: [Inference] [Speculation] [Unverified]
- Ask for clarification if information is missing. Do not guess or fill gaps

### Communication Standards
- **NO boilerplate** — Never include "Co-Authored by Claude" or AI mentions in review comments
- Review comments should read as if written by a human engineer
- Clear, professional, technically focused language

### Bash Command Rules

To avoid triggering unnecessary approval prompts:

- **No shell variable assignments** — Never write `VAR=$(command)` or `VAR=value` at the start of a Bash call. Use each command's output directly in subsequent commands as a literal value.
- **No comments before commands** — Never put `# comment` lines before or inside a Bash call. Remove all inline comments from shell commands.
- **No multi-`$()` compositions** — Never build a single command from multiple `$()` substitutions. Run each sub-command separately and use its literal output value.
- **One operation per call** — Each distinct shell operation should be its own Bash tool call.

### Reviewer-Specific Rules
- Load original story requirements first
- **Trigger and monitor CI/CD checks** — it is the reviewer's job to ensure all checks run
- **NEVER run terraform apply** — only `terraform plan` is allowed for validation
- Be critical but constructive with specific examples and file:line references
- Score objectively (1-10) with clear justification
- Verify no tests have been disabled, commented out, or mocked to always pass
- Check for debug statements (println, console.log, dbg!, etc.)

---

## Phase 1: Load PR Details

Get PR details using the actual PR number from arguments:

```bash
gh pr view 42 --json body,headRefName,number,title,isDraft
```

- Verify PR exists and is not draft
- **STOP if PR is draft** — respond: "This PR is marked as draft. Please mark it ready for review first."

---

## Phase 1.5: Detect Review Mode

Fetch all existing reviews on this PR. Use the actual PR number from arguments.

```bash
gh pr reviews 42 --json author,state,body,submittedAt
```

**Decision logic:**

- If the reviews array is empty (no reviews at all) → **First Review Mode**
- If reviews exist but none have `state: CHANGES_REQUESTED` → **First Review Mode**
- If at least one review has `state: CHANGES_REQUESTED` → **Re-Review Mode**

Output a mode banner before proceeding:

- `[FIRST REVIEW]` — you are doing the initial review; exhaustiveness is mandatory
- `[RE-REVIEW — verifying N previously requested changes]` (where N is the count of bullet items extracted from the `## Required Changes` section) — you are checking whether prior requests were addressed

**For Re-Review Mode only:**

Parse the body of the most recent `CHANGES_REQUESTED` review. Extract the bullet list under the `## Required Changes` section verbatim. This is the **Previous Required Changes** list. Carry it forward to Phase 5.

**Fallback:** If the most recent `CHANGES_REQUESTED` review body is empty, null, or does not contain a `## Required Changes` section with bullet points, log a warning: "Previous required changes not found — falling back to First Review Mode." Then treat this invocation as First Review Mode.

---

## Phase 2: Extract Story ID and Load Requirements

Parse PR body for story reference using the PM adapter's "Story Reference in PRs" format.

Also check PR title if not found in body.

**If story ID not found:**
- Post review comment requesting story ID be added
- **STOP with REQUEST_CHANGES** — do not proceed without story linkage

**Once ID found:**

1. Read `~/.claude/dev-workflow/config.json` to get `pm_adapter` and `notes_adapter`
2. Load PM adapter: check `~/.claude/skills/pm-adapter/{pm_adapter}.md` first (user override); fall back to `skills/pm-adapter/{pm_adapter}.md` → fetch story by ID
3. Detect service name: `git rev-parse --show-toplevel | xargs basename`
4. Load notes adapter: check `~/.claude/skills/notes-adapter/{notes_adapter}.md` first (user override); fall back to `skills/notes-adapter/{notes_adapter}.md` → read Claude Instructions spec
5. **If spec not found:** ERROR and ask user to run `/start writer {story-id}` first

---

## Phase 3: Trigger and Monitor CI/CD Jobs

**CRITICAL:** Ensure all CI/CD checks run before reviewing code.

**NEVER run terraform apply.** Only run terraform plan for validation.

First get the branch name for this PR:

```bash
gh pr view 42 --json headRefName -q .headRefName
```

Then list CI runs using the branch name from the output above:

```bash
gh run list --branch "branch-name-from-output" --json conclusion,status,name,createdAt,workflowName --limit 10
```

#### Required Workflows

- **Always:** build workflow (if exists), test workflow (if exists)
- **Conditionally:** terraform plan (if terraform files changed)

Trigger any missing or failed workflows. Wait up to 15 minutes for completion.

If any check fails:
- Post review comment noting the failures
- **STOP** — do not proceed with code review until checks pass

---

## Phase 4: Load PR Changes

Get the list of changed files:

```bash
gh pr view 42 --json files
```

Get the full diff:

```bash
gh pr diff 42
```

Analyze all file changes and understand the scope of modifications.

---

## Phase 5: Multi-Perspective Code Review

> **MODE CHECK:** Apply the correct review mode determined in Phase 1.5.
> - If the mode banner was `[FIRST REVIEW]` → follow **First Review Mode** below.
> - If the mode banner was `[RE-REVIEW]` → follow **Re-Review Mode** below.

### First Review Mode

> **EXHAUSTIVENESS MANDATE — FIRST REVIEW**
> This is the only opportunity to identify issues. A re-review will not accept new findings unless they meet the Critical Exception Threshold (defined below). You must find everything now:
> - Read the entire diff — every file, every line
> - Apply all four perspectives with maximum scrutiny
> - Do not defer marginal issues thinking you can raise them next time — there is no next time
> - If unsure whether something is worth flagging, flag it

Proceed with the subagent skill invocation and four-perspective analysis below.

---

### Re-Review Mode

**Do not run the subagent skill invocation or four-perspective analysis. Replace them entirely with the following:**

Work through each item in the **Previous Required Changes** list extracted in Phase 1.5. For each item:

- **Status:** Addressed / Not Addressed / Partially Addressed
- **Evidence:** Specific `file:line` reference from the current diff showing the change (or its absence)

Items with **Status: Not Addressed** or **Partially Addressed** must be carried forward verbatim into the Required Changes section of the Phase 6 review body. If a required change was addressed by deleting the problematic code entirely, mark it **Addressed** with Evidence citing the deletion.

After verifying all previous items, evaluate whether any **new** findings meet the Critical Exception Threshold:

> **Critical Exception Threshold** — A new finding qualifies ONLY if ALL of the following are true:
> 1. It is a security vulnerability, data loss risk, or correctness failure that would cause incorrect behavior in production
> 2. It was introduced by code added **after** the original review — it does not appear in the original diff
> 3. It is not present in the original diff in any form — not as the same code, a functionally equivalent pattern, or a related code path. If it appears anywhere in the original diff, it does not qualify regardless of how subtle or easy to overlook it was.
>
> If a finding does not satisfy all three criteria, it must be omitted. Do not rationalize marginal findings into this threshold.

List qualifying new findings under a **New Critical Findings** section with an explicit per-item justification of why the threshold is met.

Once the verification checklist is complete, skip to Phase 5.5.

> **STOP — Re-Review Mode ends here. Do not continue reading Phase 5. Proceed directly to Phase 5.5.**

---

### First Review Mode (continued)

Before beginning the perspective-based review, invoke the code review request skill:

> Invoke Skill: `superpowers:requesting-code-review`
>
> Provide the code-reviewer subagent with:
> - The Claude Instructions spec loaded in Phase 2 as the expected-functionality reference
> - The story acceptance criteria from Phase 2
> - The PR diff from Phase 4 as the code under review
>
> After the subagent completes, incorporate its findings into the four-perspective analysis
> below. Do not duplicate findings — use subagent output to inform and enrich each section.

Review from four perspectives sequentially, comparing against: story requirements, Claude Instructions, and PR diff.

#### A. Product Manager Review (Feature Completeness)
- Does the implementation match the acceptance criteria?
- Does it solve the original user problem?
- Are there UX concerns?
- Are there gaps between what was requested and delivered?

#### B. Principal Developer Review (Code Quality)
- Are there bugs or logic errors?
- Is error handling comprehensive?
- Are there debug statements or commented-out code that shouldn't be there?
- Does it follow existing codebase patterns?
- Are there any obvious performance issues?
- Are there security vulnerabilities (injection, auth bypass, etc.)?

#### C. QA Engineer Review (Test Coverage)
- Are there sufficient tests for the new functionality?
- Are any tests disabled, skipped, or mocked to always pass?
- Do tests cover error cases and edge cases?
- Would these tests have caught the bug (for bug fixes)?

#### D. Software Architect Review (Structure)
- Is the code over-engineered for this use case?
- Are new abstractions actually needed?
- Does the structure align with the existing architecture?
- Are there any concerning dependencies introduced?

---

## Phase 5.5: Verify Before Submitting Review

> Invoke Skill: `superpowers:verification-before-completion`
>
> Verify with fresh command execution:
> - All CI/CD checks still pass (re-run status check from Phase 3)
> - The PR diff reflects current HEAD (no new commits since loading in Phase 4)
> - **First Review Mode:** No required changes identified in Phase 5 have been overlooked
> - **Re-Review Mode:** All unresolved items from the Phase 5 verification checklist have been carried forward to Phase 6. No New Critical Findings that meet the exception threshold have been omitted.

---

## Phase 6: Synthesize and Submit Review

**Scoring (1-10):**
- 9-10: Excellent, approve with minor optional suggestions
- 7-8: Good, approve with minor required fixes noted
- 5-6: Adequate but has issues that should be addressed
- 3-4: Significant problems, request changes
- 1-2: Fundamental issues, request changes with priority concerns

*In Re-Review Mode: score is based primarily on how completely previous required changes were addressed. New critical findings that meet the exception threshold lower the score as in first review.*

**For Re-Review Mode**, use this body format instead:

~~~markdown
## Review Score: X/10

## Summary
[2-3 sentence overview of whether previous requests were addressed]

## Previous Changes Verification
- [original required change item]: Addressed — `file.ts:42` resolves the issue
- [original required change item]: Not Addressed — no relevant change found in diff
- [original required change item]: Partially Addressed — `file.ts:88` handles the happy path but the error case remains

## New Critical Findings (Exception Threshold Met)
[Omit this section entirely if no new findings qualify. If present, include per-item threshold justification for each finding.]

## Required Changes
[Unresolved items from Previous Changes Verification only, plus any qualifying New Critical Findings. No other new items are permitted in re-review.]

## Suggestions (Optional)
[Omit entirely in re-review unless directly related to an unresolved previous request]
~~~

**For First Review Mode**, use the original body format:

**Review body format:**
```markdown
## Review Score: X/10

## Summary
[2-3 sentence overview of the implementation quality]

## Product Manager Perspective
[Feature completeness findings]

## Developer Perspective
[Code quality findings with file:line references]

## QA Perspective
[Test coverage findings]

## Architect Perspective
[Structure findings]

## Required Changes
[Bulleted list of must-fix items]

## Suggestions (Optional)
[Bulleted list of nice-to-have improvements]
```

Submit formal GitHub review with decision:
- **APPROVE** if score ≥ 8 and no required changes
- **REQUEST_CHANGES** if required changes exist
- **COMMENT** if neutral/informational

Use the actual PR number:

```bash
gh pr review 42 --approve --body "..."
```
