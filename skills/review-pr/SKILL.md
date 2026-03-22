---
name: review-pr
description: "Comprehensive multi-perspective PR review (Product Manager, Developer, QA, and Architect lenses) that compares implementation against story requirements and CI/CD results. Includes automatic first-review vs re-review mode detection. Always use this when a user asks to review a PR, check a pull request, or validate an implementation against requirements."
---

# Review PR

**Role:** Review PR — comprehensive PR review comparing implementation against story requirements

## Arguments: $ARGUMENTS

Either a PR number (e.g., `42`) or a PM ticket ID (e.g., `sc-123`) may be passed as the argument.

Read `skills/shared/standards.md` — these mandatory rules govern this entire session.

---

## Phase 0: Resolve Input to PR Number

Parse the argument from `$ARGUMENTS`.

**If the argument is purely numeric** → it is a PR number. Use it directly. Skip to Phase 1.

**If the argument contains non-numeric characters** (e.g., `sc-123`, `LIN-456`) → treat it as a PM ticket ID and resolve it:

1. Read `~/.claude/dev-workflow/config.json` to get `pm_adapter`
2. Load PM adapter (user override first, then built-in)
3. Use the adapter's **"Finding PRs linked to a story"** instructions to look up linked PRs — adapters with native API support (Shortcut, Jira, GitHub Issues) will return authoritative results; others fall back to `gh pr list --state all --search "{story_id}"`
4. **If exactly one PR is found** → extract its number. Use it as the PR number for all subsequent phases.
5. **If no PRs are found** → STOP: "No PR found referencing {story_id}. Ensure the PR is linked to the story in the format your PM adapter expects, then try again."
6. **If multiple PRs are found** → list them (number, title, state) and ask the user: "Multiple PRs reference {story_id}. Which PR number should I review?"

---

## CRITICAL: Reviewer-Specific Rules
- Load original story requirements first
- **Trigger and monitor CI/CD checks** — it is the reviewer's job to ensure all checks run
- **NEVER run terraform apply** — only `terraform plan` is allowed for validation
- Be critical but constructive with specific examples and file:line references
- Score objectively (1-10) with clear justification
- Verify no tests have been disabled, commented out, or mocked to always pass
- Check for debug statements (println, console.log, dbg!, etc.)

---

> **Note:** In all bash examples below, `{PR_NUMBER}` is a placeholder. Replace it with the actual PR number from `$ARGUMENTS` in every command you run.

## Phase 1: Load PR Details

Get PR details using the actual PR number from arguments:

```bash
gh pr view {PR_NUMBER} --json body,headRefName,number,title,isDraft
```

- Verify PR exists and is not draft
- **STOP if PR is draft** — respond: "This PR is marked as draft. Please mark it ready for review first."

---

## Phase 1.5: Detect Review Mode

Fetch all existing reviews on this PR. Use the actual PR number from arguments.

```bash
gh pr reviews {PR_NUMBER} --json author,state,body,submittedAt
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
5. **If spec not found:** ERROR and ask user to invoke the Writer skill (`dev-workflow:write-spec`) with this story ID first

---

## Phase 3: Trigger and Monitor CI/CD Jobs

**CRITICAL:** Ensure all CI/CD checks run before reviewing code.

**NEVER run terraform apply.** Only run terraform plan for validation.

First get the branch name for this PR:

```bash
gh pr view {PR_NUMBER} --json headRefName -q .headRefName
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
gh pr view {PR_NUMBER} --json files
```

Get the full diff:

```bash
gh pr diff {PR_NUMBER}
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
> - Apply all six specialist perspectives with maximum scrutiny
> - Do not defer marginal issues thinking you can raise them next time — there is no next time
> - If unsure whether something is worth flagging, flag it

**Step 1:** Read all six subagent instruction files **in parallel**:
- `skills/subagents/code-quality-engineer.md`
- `skills/subagents/simple-architecture-engineer.md`
- `skills/subagents/security-engineer.md`
- `skills/subagents/performance-engineer.md`
- `skills/subagents/product-manager.md`
- `skills/subagents/budget-manager.md`

**Step 2:** Launch all 6 review subagents **in parallel** (all in a single message with multiple Agent tool calls).

For each agent, craft a prompt that embeds:
1. The full content of its instruction file (read in Step 1)
2. The following shared context block:

```
---
STORY REQUIREMENTS:
[Story title, description, and all acceptance criteria from Phase 2]

SPECIFICATION (Claude Instructions):
[Full spec content from Phase 2]

CI/CD STATUS:
[Build/test/terraform results from Phase 3]

PR DIFF:
[Full diff from Phase 4]
---
```

Launch the following 6 agents in parallel:

1. **Code Quality Engineer** (subagent_type: general-purpose)
   - Instructions: content of `skills/subagents/code-quality-engineer.md` + shared context
   - Focus: correctness, naming, test quality, debug artifacts, consistency

2. **Simple Architecture Engineer** (subagent_type: general-purpose)
   - Instructions: content of `skills/subagents/simple-architecture-engineer.md` + shared context
   - Focus: DRY, YAGNI, over-engineering, unnecessary abstractions, new dependencies

3. **Security Engineer** (subagent_type: general-purpose)
   - Instructions: content of `skills/subagents/security-engineer.md` + shared context
   - Focus: OWASP Top 10, injection, auth, sensitive data exposure, input validation

4. **Performance Engineer** (subagent_type: general-purpose)
   - Instructions: content of `skills/subagents/performance-engineer.md` + shared context
   - Focus: N+1 queries, algorithm complexity, memory, caching, async blocking

5. **Product Manager** (subagent_type: general-purpose)
   - Instructions: content of `skills/subagents/product-manager.md` + shared context
   - Focus: acceptance criteria verification, user problem fit, regressions, readiness

6. **Budget Manager** (subagent_type: general-purpose)
   - Instructions: content of `skills/subagents/budget-manager.md` + shared context
   - Focus: external API costs, infrastructure, storage, scaling cost profile

Wait for all 6 agents to complete, then collect all findings before proceeding to Phase 5.5.

---

### Re-Review Mode

**Do not run the parallel subagents. Replace them entirely with the following:**

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

**For First Review Mode**, synthesize all six subagent reports into this format. De-duplicate findings that appear across multiple reports — each issue should appear once, in the section most relevant to it. Prioritize findings by severity when populating Required Changes.

**Review body format:**
```markdown
## Review Score: X/10

## Summary
[2-3 sentence overview of the implementation quality]

## Code Quality
[Key findings from Code Quality Engineer — correctness, naming, test coverage]

## Architecture
[Key findings from Simple Architecture Engineer — simplicity, DRY, design]

## Security
[Key findings from Security Engineer — omit section if no findings]

## Performance
[Key findings from Performance Engineer — omit section if no findings]

## Product Fit
[Acceptance criteria verification and product findings from Product Manager]

## Budget Impact
[Cost impact rating and findings from Budget Manager — omit section if no new costs]

## Required Changes
[Consolidated must-fix items from all reviewers, bulleted with file:line references]

## Suggestions (Optional)
[Consolidated nice-to-have improvements from all reviewers]
```

Submit formal GitHub review with decision:
- **APPROVE** if score ≥ 8 and no required changes
- **REQUEST_CHANGES** if required changes exist
- **COMMENT** if neutral/informational

Use the actual PR number:

```bash
gh pr review {PR_NUMBER} --approve --body "..."
```
