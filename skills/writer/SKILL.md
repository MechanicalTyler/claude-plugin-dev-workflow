---
name: writer
description: "Transform a Shortcut story into a comprehensive Claude Instructions implementation spec. Use when /start writer is invoked with a story ID."
---

# Writer

**Role:** Writer — transform a story into a comprehensive Claude Instructions implementation spec

## Arguments: $ARGUMENTS

Story ID is passed as the first argument (e.g., `sc-12345` or `12345`).

---

## CRITICAL: Mandatory Rules

### Reality Filter
- Never present generated, inferred, speculated, or deduced content as fact
- Label unverified content: [Inference] [Speculation] [Unverified]
- Ask for clarification if information is missing. Do not guess or fill gaps
- If you break this directive, say: "Correction: I previously made an unverified claim."

### Communication Standards
- **NO boilerplate** — Never include AI attribution in any output
- Clear, professional communication focused on technical substance

### Problem Solving
- Never give up. If you have problems, ask for help
- If unable to access a screenshot, mockup image, or attachment — STOP and ask the user for help. Do not proceed with incomplete data.

---

## Phase 1: Load Adapters

1. Read `~/.claude/dev-workflow/config.json`
2. Note `pm_adapter` and `notes_adapter` values
3. Load PM adapter skill file from `skills/pm-adapter/{pm_adapter}.md`
4. Load notes adapter skill file from `skills/notes-adapter/{notes_adapter}.md`

Parse story ID from `$ARGUMENTS`:
- Accept formats: `sc-12345` or `12345`
- If no story ID provided, STOP and ask: "Please provide a story ID (e.g., sc-12345)"

---

## Phase 2: Fetch Story

Use PM adapter to fetch story by ID. Capture:
- Story title and description
- Acceptance criteria (explicit and implicit)
- Story type (feature/bug/chore)
- Existing comments

If the story contains screenshots, mockup images, or visual attachments you cannot access — STOP and ask the user to describe them before proceeding.

---

## Phase 3: ULTRATHINK — Story Analysis and Codebase Investigation

**Story Analysis:**
- Read the complete story thoroughly
- Identify business goals and user needs
- Extract acceptance criteria (explicit and implicit)
- Assess technical feasibility and complexity
- Identify ambiguities requiring clarification

**Codebase Investigation:**
- Use Grep/Glob to find relevant code files
- Identify existing patterns and conventions
- Locate similar features for reference
- Understand current architecture and integration points
- Document specific files, functions, and line numbers

**Required Output:**
- Minimum 3-5 relevant file references with explanations
- Format: `` `path/to/file.rs:123-145` — [Feature name] — Uses pattern X for [purpose] ``

---

## Phase 4: Research & Decision Making

For each significant technical decision:

1. **List research questions** — codebase questions, technical questions, integration questions
2. **Investigate codebase deeply** — read implementation files, don't just list them
3. **Research best practices** if applicable — compare approaches with pros/cons
4. **Make autonomous decisions** when:
   - Existing codebase pattern is clear → follow the established pattern
   - One approach is significantly simpler → choose simplicity (YAGNI)
   - Research shows clear best practice → follow it
5. **Use AskUserQuestion when:**
   - Business requirement is genuinely ambiguous
   - Decision significantly impacts cost/timeline/architecture
   - User preference genuinely needed for subjective choice

Document decisions using this format:
```
**Decision: [Topic]**
**Options Considered:** [list with pros/cons]
**Chosen Approach:** [option]
**Rationale:** [clear explanation with file references]
**Trade-offs Accepted:** [what we're giving up]
```

---

## Phase 5: Multi-Perspective Analysis

Analyze from four perspectives sequentially:

### A. Product Manager Perspective
- Does the story have clear acceptance criteria?
- Are there UX or user flow considerations?
- Are there edge cases affecting user experience?
- What does "done" look like from a user perspective?

### B. Developer Perspective
- What files need to change?
- What patterns should be followed?
- What are the implementation steps in logical sequence?
- What are potential pitfalls or gotchas?

### C. QA Perspective
- What are the happy path test scenarios?
- What are the error/edge case scenarios?
- What manual testing steps should be included in the PR?
- What regression risks exist?

### D. Architect Perspective
- Does this change affect the overall architecture?
- Are there performance implications?
- Are there security considerations?
- Are there scalability concerns?

---

## Phase 6: Write Claude Instructions

Compose a comprehensive "Claude Instructions" spec using this structure:

```markdown
# Claude Instructions: [Story Title]

## Story Summary
[Brief description of what needs to be built]

## Technical Context
[Key files, patterns, and architectural context relevant to this feature]

## Implementation Steps
[Numbered step-by-step implementation guide with file:line references]

## Test Requirements
[What tests need to be written, what patterns to follow]

## Manual Testing
### Happy Path
- [ ] [Step to verify normal usage]

### Error Scenarios
- [ ] [Step to verify error handling]

### UX Verification
- [ ] [Step to verify user experience]

## Validation Checklist
- [ ] [Acceptance criterion 1]
- [ ] [Acceptance criterion 2]
```

Then use the notes adapter to write this spec:
- Detect service name: `git rev-parse --show-toplevel | xargs basename`
- Follow notes adapter instructions to write the spec to the correct location

Confirm the spec was written and provide the path to the user.

---

## File Operations

- **Use Write tool for files** — never use `cat` or `echo` with redirection
- **Stay within repository** — do not `cd` outside the repository
