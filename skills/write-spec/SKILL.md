---
name: dev-workflow:write-spec
description: "Use when a developer needs a detailed technical spec before coding, when a user provides a story ID and asks for a spec, implementation plan, or Claude Instructions, or always before the Start Development skill when working from a PM story."
---

# Write Spec

**Role:** Write Spec — transform a story into a comprehensive Claude Instructions implementation spec

**SCOPE BOUNDARY:** This skill writes a spec file and NOTHING else. It does **not** write code, write any other local files, make commits, checkout git branches, implement features, or begin development. When the spec file is saved, output the path and STOP.

**AUDIENCE:** The spec must be digestible by a product manager. Write in plain language describing *what* to build and *why*, not *how* to code it. Implementation details like code examples, function signatures, and algorithmic pseudocode are the developer's responsibility — omit them from the spec.

## Arguments: $ARGUMENTS

Story ID is passed as the first argument (e.g., `sc-12345` or `12345`).

Read `skills/shared/standards.md` — these mandatory rules govern this entire session.

Read `skills/shared/adapter-loading.md` — adapter loading procedures referenced in Phase 1.

---

## Phase 1: Load Adapters

1. Read `~/.claude/dev-workflow/config.json`
2. Note `pm_adapter` and `notes_adapter` values
3. Load PM adapter per procedure in `skills/shared/adapter-loading.md`
4. Load notes adapter per procedure in `skills/shared/adapter-loading.md`

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
> Focus on: gaps or contradictions in the stated acceptance criteria,
> and architectural questions within the stated scope.
>
> OVERRIDE: After brainstorming completes, do NOT invoke `superpowers:writing-plans`.
> Return to Phase 5 (ULTRATHINK) — the brainstorming output informs that analysis.

**Interactive mode — full-understanding mandate:** When running in interactive mode (you can ask the user questions), it is your job to fully understand the entire feature before writing the spec. Keep asking the user clarifying questions — in a back-and-forth — until no material ambiguity about scope, behavior, edge cases, or intent remains. A spec you do not fully understand is a spec you cannot write correctly. This overrides the general "questions are a last resort" posture in `standards.md` *for spec writing specifically*: still investigate the codebase first, but where investigation cannot settle a question and you are interactive, **ask rather than assume**. Do not move past this phase with an understanding you would describe as partial.

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
5. **Asking the user:**
   - **Interactive mode:** Per the full-understanding mandate in Phase 4, ask whenever investigation cannot resolve a question that affects scope, behavior, edge cases, or intent — the high-stakes bar does not apply to spec writing. Always investigate the codebase, docs, and git history first; ask about what investigation genuinely cannot settle. Do not leave a question unresolved by choosing not to ask.
   - **Autonomous mode (no AskUserQuestion available):** You cannot ask. Resolve every question by investigation. Where investigation cannot fully settle a question, make the most reasonable decision, label it `[Inference]` with rationale, and proceed. Never emit an `[Open Question]` — see Phase 7.

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

**No open questions in the final spec.** A finished spec must never contain `[Open Question]` items. Every question must be resolved before the spec is written — either automatically through investigation, or, in interactive mode, through back-and-forth with the user. Resolve them; do not defer them.

**Decision:**
- **Interactive mode** — If any criterion cannot be met after research, do not write a partial spec. Take the unresolved gaps back to the user and resolve them in a back-and-forth (re-enter the Phase 4 brainstorm if the gaps are substantial). Only proceed to write the spec once every criterion is met and no open questions remain:
  > "Before I can write a complete spec, I need to resolve these with you:
  > - [list specific gaps]
  >
  > [Ask the specific questions needed to close each gap.]"
- **Autonomous mode (cannot ask)** — Resolve every gap by investigation. For any gap investigation cannot fully close, make the most reasonable decision, document it as a `[Decision]` with `[Inference]`-labeled rationale and the trade-off, and proceed. Never emit `[Open Question]` — a documented inferred decision is required instead.
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
> Phase 10 spec writing only.
>
> OVERRIDE: Implementation steps must describe WHAT to do, not HOW to code it.
> Use plain language (e.g., "Add a validation endpoint that checks X against Y")
> not code examples or pseudocode. The developer determines the code.

---

## Phase 10: Write Claude Instructions

**Spec writing rules:**
- Describe behavior and requirements, not code. No code blocks, pseudocode, or function signatures.
- File references (e.g., `path/to/file.rs`) are acceptable for pointing developers to the right location. Code excerpts from those files are not.
- A product manager should be able to read this spec and understand every section.

The spec is a human-readable artifact saved to a local file, so it must be a **standalone HTML document** per the Output Format rules in `skills/shared/standards.md` — not markdown. Compose a comprehensive "Claude Instructions" spec using this structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Claude Instructions: [Story Title]</title>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 56rem; margin: 2rem auto; padding: 0 1.5rem; line-height: 1.6; color: #1a1a1a; }
    h1 { border-bottom: 2px solid #ddd; padding-bottom: .3rem; }
    h2 { margin-top: 2rem; }
    code { background: #f4f4f4; padding: .1rem .3rem; border-radius: 3px; font-family: ui-monospace, monospace; }
    ul { padding-left: 1.4rem; }
    .check { list-style: none; padding-left: 0; }
    .check li::before { content: "\2610  "; }
  </style>
</head>
<body>
  <h1>Claude Instructions: [Story Title]</h1>

  <h2>Story Summary</h2>
  <p>[Brief description of what needs to be built]</p>

  <h2>Technical Context</h2>
  <p>[Key files, patterns, and architectural context relevant to this feature]</p>

  <h2>Implementation Steps</h2>
  <ol>
    <li>[Step with <code>file:line</code> references]</li>
  </ol>

  <h2>Test Requirements</h2>
  <p>[What tests need to be written, what patterns to follow]</p>

  <h2>Manual Testing</h2>
  <h3>Happy Path</h3>
  <ul class="check"><li>[Step to verify normal usage]</li></ul>
  <h3>Error Scenarios</h3>
  <ul class="check"><li>[Step to verify error handling]</li></ul>
  <h3>UX Verification</h3>
  <ul class="check"><li>[Step to verify user experience]</li></ul>

  <h2>Validation Checklist</h2>
  <ul class="check">
    <li>[Acceptance criterion 1]</li>
    <li>[Acceptance criterion 2]</li>
  </ul>
</body>
</html>
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
  > Spec: `docs/specs/{service-name}/{story-id}.html`
  > Covers: [1-2 sentence summary of implementation approach]
  > Written by: Claude Write Spec workflow

If the PM adapter supports attaching files or adding external links, prefer that over a comment.

If the PM adapter does not support comments or updates — note this to the user and provide the spec path so they can link it manually.

---

## Phase 12: Adversarial Review

Read and follow the adversarial review procedure in `skills/shared/adversarial-review.md` with these context variables:

- `story_id`: the story ID from `$ARGUMENTS`
- `review_target`: "spec document"
- `review_context`: the full Claude Instructions spec content written in Phase 10

The adversarial agent verifies the spec completely and accurately captures all story requirements, acceptance criteria, and testing instructions. It checks that no story requirements were dropped, watered down, or misinterpreted.

---

