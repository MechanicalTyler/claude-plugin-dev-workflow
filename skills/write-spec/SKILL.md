---
name: write-spec
description: "Use when a developer needs a detailed technical spec before coding, when a user provides a story ID and asks for a spec, implementation plan, or Claude Instructions, or always before the Start Development skill when working from a PM story."
---

# Write Spec

**Role:** Write Spec — transform a story into a comprehensive Claude Instructions implementation spec

**SCOPE BOUNDARY:** This skill writes a spec file and NOTHING else. It does **not** write code, write any other local files, make commits, checkout git branches, implement features, or begin development. When the spec file is saved, output the path and STOP.

## Arguments: $ARGUMENTS

Story ID is passed as the first argument (e.g., `sc-12345` or `12345`).

Read `skills/shared/standards.md` — these mandatory rules govern this entire session.

---

## Phase 1: Load Adapters

1. Read `~/.claude/dev-workflow/config.json`
2. Note `pm_adapter` and `notes_adapter` values
3. Load PM adapter: check `~/.claude/skills/pm-adapter/{pm_adapter}.md` first (user override); fall back to `skills/pm-adapter/{pm_adapter}.md`
4. Load notes adapter: check `~/.claude/skills/notes-adapter/{notes_adapter}.md` first (user override); fall back to `skills/notes-adapter/{notes_adapter}.md`

Parse story ID from `$ARGUMENTS`:
- Accept formats: `sc-12345` or `12345`
- If no story ID provided, STOP and ask: "Please provide a story ID (e.g., sc-12345)"

---

## Phase 2: Check for Existing Spec

Use the notes adapter to check whether a spec already exists for this story ID.

- If an **existing spec is found**: STOP and ask the user:
  > "A spec already exists for [story-id] at [path]. Would you like to:
  > 1. Use it as additional context and continue writing a new spec
  > 2. Update/overwrite the existing spec
  > 3. Cancel — keep the existing spec unchanged"
  >
  > Wait for the user to choose an option before proceeding. If you are unable to ask the user (e.g. running non-interactively), notify them and skip:
  > "A spec already exists for [story-id] at [path]. Skipping spec creation."

- If **no existing spec is found**: continue to Phase 2.

---

## Phase 3: Fetch Story

Use PM adapter to fetch story by ID. Capture:
- Story title and description
- Acceptance criteria (explicit and implicit)
- Story type (feature/bug/chore)
- Existing comments

If the story contains screenshots, mockup images, or visual attachments you cannot access — STOP and ask the user to describe them before proceeding.

---

## Phase 4: Brainstorm Ambiguities and Approach

Before the ULTRATHINK deep-dive, invoke brainstorming to surface unclear requirements:

> Invoke Skill: `superpowers:brainstorming`
>
> Focus on: acceptance criteria ambiguities, implicit requirements not stated in the story,
> and architectural questions.
>
> OVERRIDE: After brainstorming completes, do NOT invoke `superpowers:writing-plans`.
> Return to Phase 5 (ULTRATHINK) — the brainstorming output informs that analysis.

---

## Phase 5: ULTRATHINK — Story Analysis and Codebase Investigation

**SCOPE BOUNDARY: You are writing a spec for THIS service only.**
- Detect the current service: `git rev-parse --show-toplevel | xargs basename`
- All codebase research MUST target files in the current repository
- Other services or systems mentioned in the story are **reference context only** — do not write implementation steps, file changes, or instructions for them
- If the story requires coordinated changes across services, note it as: `[Cross-service dependency: {service-name} — out of scope for this spec]`

**Story Analysis:**
- Read the complete story thoroughly
- Identify business goals and user needs
- Extract acceptance criteria (explicit and implicit)
- Assess technical feasibility and complexity
- Identify ambiguities requiring clarification

**Codebase Investigation (this service only):**
- Use Grep/Glob to find relevant code files in the current repository
- Identify existing patterns and conventions
- Locate similar features for reference
- Understand current architecture and integration points
- Document specific files, functions, and line numbers

**Required Output:**
- Minimum 3-5 relevant file references with explanations (all from THIS service)
- Format: `` `path/to/file.rs:123-145` — [Feature name] — Uses pattern X for [purpose] ``

---

## Phase 6: Research & Decision Making

For each significant technical decision:

1. **List research questions** — codebase questions, technical questions, integration questions
2. **Investigate codebase deeply** — read implementation files, don't just list them; trace call paths, read tests, check git blame for intent
3. **Research best practices** if applicable — compare approaches with pros/cons
4. **Make autonomous decisions** — default to deciding. You should make decisions in the vast majority of cases:
   - Existing codebase pattern is clear → follow the established pattern
   - One approach is significantly simpler → choose simplicity (YAGNI)
   - Research shows clear best practice → follow it
   - Ambiguous but not high-stakes → pick the most reasonable option, document the trade-off, and move on
5. **Use AskUserQuestion ONLY when ALL of these are true:**
   - The answer cannot be found by reading the codebase, docs, or git history
   - The decision is genuinely high-stakes (significantly impacts cost, timeline, or architecture)
   - Getting it wrong would require substantial rework — not just a minor tweak

Document decisions using this format:
```
**Decision: [Topic]**
**Options Considered:** [list with pros/cons]
**Chosen Approach:** [option]
**Rationale:** [clear explanation with file references]
**Trade-offs Accepted:** [what we're giving up]
```

---

## Phase 7: Go/No-Go Spec Decision

After deep research, assess whether you have enough information to write a spec that a junior developer could implement without significant guesswork.

**Evaluate against these criteria:**
- [ ] You can name the specific files that need to change
- [ ] You understand the existing patterns to follow
- [ ] Acceptance criteria can be mapped to concrete implementation steps
- [ ] Edge cases and error states are understood or clearly deferrable
- [ ] No critical unknowns remain that would block implementation

**Decision:**
- If **3 or more criteria cannot be met** even after research — STOP and ask the user:
  > "After researching the codebase, I'm missing information needed to write a complete spec:
  > - [list specific gaps]
  >
  > Would you like to provide this context, or shall I write a partial spec and flag these as `[Open Question]` items?"
- If **minor gaps remain** — proceed and document them as `[Open Question]` items in the spec.
- If **all criteria are met** — proceed immediately.

---

## Phase 8: Multi-Perspective Analysis

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

## Phase 9: Structure Implementation Tasks

Before writing the final spec, use the writing-plans methodology to structure the implementation steps with granularity:

> Invoke Skill: `superpowers:writing-plans`
>
> OVERRIDE: Do NOT save a separate plan file. Use the task breakdown produced here as
> the content for the "Implementation Steps" section of the Claude Instructions spec in Phase 10.
>
> OVERRIDE: Do NOT offer execution options at the end of this invocation. Output feeds into
> Phase 6 spec writing only.

---

## Phase 10: Write Claude Instructions

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

## Phase 11: Link Spec to PM Ticket

After the spec file is saved, update the original PM story to reference it:

Use the PM adapter to add a comment or description update to the story. The comment should include:
- The spec file path (relative to the repo root)
- A brief summary of what the spec covers
- Example format:
  > **Implementation Spec Written**
  > Spec: `docs/specs/{service-name}/{story-id}.md`
  > Covers: [1-2 sentence summary of implementation approach]
  > Written by: Claude Write Spec workflow

If the PM adapter supports attaching files or adding external links, prefer that over a comment.

If the PM adapter does not support comments or updates — note this to the user and provide the spec path so they can link it manually.

---

