# Reviewer: Review Mode Detection Design

**Date:** 2026-03-04
**Topic:** Prevent back-and-forth review loops by detecting first vs re-review and enforcing appropriate behavior in each mode

---

## Problem

The reviewer agent was finding new issues on each re-review rather than identifying everything comprehensively on the first pass. This created a back-and-forth loop where developers would address one batch of issues only to receive a new batch in the next review cycle.

---

## Solution Overview

Add a **Review Mode Detection** phase (Phase 1.5) early in the reviewer flow. This phase determines whether the current invocation is a **First Review** or a **Re-Review** by inspecting existing GitHub reviews on the PR. Phase 5 (Multi-Perspective Code Review) then branches based on mode.

---

## Phase 1.5: Review Mode Detection

Inserted between Phase 1 (Load PR Details) and Phase 2 (Extract Story ID).

Fetch all existing reviews:

```bash
gh pr reviews {PR} --json author,state,body,submittedAt
```

**Decision logic:**
- No reviews with `state: REQUEST_CHANGES` → **First Review Mode**
- At least one `REQUEST_CHANGES` review exists → **Re-Review Mode**

For re-review mode: parse the most recent REQUEST_CHANGES review body and extract the "Required Changes" bullet list verbatim. This becomes the **Previous Required Changes** list used in Phase 5.

Output a mode banner before continuing:
- `[FIRST REVIEW]` — proceed with exhaustiveness mandate
- `[RE-REVIEW — verifying N previously requested changes]` — proceed with verification mode

---

## Phase 5 — First Review Mode

A mandatory preamble is prepended before the four-perspective analysis:

> **EXHAUSTIVENESS MANDATE — FIRST REVIEW**
> This is the only opportunity to identify issues. A re-review will not accept new findings unless they meet the critical exception threshold. You must now find everything:
> - Read the entire diff — every file, every line
> - Apply all four perspectives with maximum scrutiny
> - Do not defer marginal issues thinking you can catch them next time — there is no next time
> - If unsure whether something is worth flagging, flag it

The four-perspective analysis (PM, Developer, QA, Architect) runs unchanged after the mandate.

---

## Phase 5 — Re-Review Mode

The four-perspective analysis is **replaced entirely** with a structured verification against the Previous Required Changes list.

For each previous required change:
- **Status**: Addressed / Not Addressed / Partially Addressed
- **Evidence**: Specific file:line reference from the new diff

### Critical Exception Threshold

New findings may only be raised in re-review if ALL of the following are true:
1. It is a security vulnerability, data loss risk, or correctness failure that would cause incorrect behavior in production
2. It was introduced by code added **after** the original review (not present in the original diff)
3. It could not reasonably have been identified from the original diff

Any finding that does not meet all three criteria must be omitted.

---

## Phase 6 Review Body — Additional Section

The re-review body format includes a new section:

```markdown
## Previous Changes Verification
- [item]: Addressed — `file.ts:42` now validates input correctly
- [item]: Not Addressed — no change found related to error handling
- [item]: Partially Addressed — ...

## New Critical Findings (Exception Threshold Met)
[Only present if applicable, with threshold justification per item]
```

The Required Changes section is populated **only** from:
- Unresolved items from the Previous Required Changes list
- New findings that explicitly meet the Critical Exception Threshold

No other new required changes are permitted in re-review.

---

## Implementation Changes

- `skills/reviewer/SKILL.md`: Insert Phase 1.5, modify Phase 5 with mode branching, update Phase 6 format
- No new files required
