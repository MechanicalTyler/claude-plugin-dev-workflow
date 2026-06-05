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

**This phase runs per repo** (see Phase 3 for repo discovery). For each repo in scope, use the notes adapter to check whether a spec already exists for this story ID in that repo's spec location.

- If an **existing spec is found** for a repo: STOP and ask the user:
  > "A spec already exists for [story-id] in repo [repo-name] at [path]. Would you like to:
  > 1. Use it as additional context and continue writing a new spec
  > 2. Update/overwrite the existing spec
  > 3. Skip this repo — keep the existing spec unchanged"
  >
  > Wait for the user to choose an option before proceeding for that repo. If you are unable to ask the user (e.g. running non-interactively), notify them and skip:
  > "A spec already exists for [story-id] in repo [repo-name] at [path]. Skipping spec creation for this repo."

- If **no existing spec is found**: continue to Phase 3.

---

## Phase 3: Fetch Story and Determine Repos

Use PM adapter to fetch story by ID. Capture:
- Story title and description
- Acceptance criteria (explicit and implicit)
- Story type (feature/bug/chore)
- Existing comments
- **"Repos to modify"** field — a comma-joined list of repo/service names (e.g. `api, web, worker`)

If the story contains screenshots, mockup images, or visual attachments you cannot access — STOP and ask the user to describe them before proceeding.

### Repo Discovery (mirrors create-story Phase 0)

Run `git rev-parse --show-toplevel` to determine context:

- **Inside a single git repo:** operate on that repo only. The "Repos to modify" field is informational; behave exactly as today (single-repo path, no loop). This preserves full backward compatibility.
- **Not inside a git repo (parent/workspace folder):** glob `{CWD}/*/.git` to discover repos. Each immediate sub-folder containing `.git` is a candidate repo. Cross-reference this list against the story's "Repos to modify" field:
  - If "Repos to modify" is present: use only the listed repos (matched by folder name or service name).
  - If "Repos to modify" is absent or empty: use all discovered repos.

**Per-item repo tags:** Acceptance criteria and testing-instruction items may be prefixed with a bracketed repo marker (e.g. `[api]`, `[web]`). Items tagged `[all]` or untagged apply to all repos. Use these tags in Phase 5–10 to filter scope per repo.

**Single-repo shortcut:** When exactly one repo is in scope (either because you are inside a git repo, or the story names exactly one repo), skip the per-repo loop entirely and proceed with existing single-repo behavior unchanged.

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

## Phases 5–10: Per-Repo Sequential Loop

> **When multiple repos are in scope** (see Phase 3 repo discovery), run Phases 5–10 once for each repo in the "Repos to modify" list, sequentially, driven by the main agent (NO sub-agents). Complete all phases for repo N before starting repo N+1.
>
> For each iteration:
> - Set the active repo root to that repo's directory.
> - Filter acceptance criteria and testing instructions to items tagged `[{repo-name}]`, `[all]`, or untagged.
> - The notes adapter resolves `repo_root` to the repo currently being specced.
> - On completion of the loop, proceed to the **User Approval Gate** (after Phase 10) before Phase 11.
>
> **When only one repo is in scope**, execute Phases 5–10 once with no loop overhead — existing behavior unchanged.

---

## Phase 5: ULTRATHINK — Story Analysis and Codebase Investigation

**SCOPE BOUNDARY: You are writing a spec for the repo currently being specced.**
- Detect the active repo: `git -C {repo_root} rev-parse --show-toplevel | xargs basename`
- All codebase research MUST target files in the repo currently being specced
- Other repos or services listed in "Repos to modify" will each get their own spec in their own loop iteration — do not write implementation steps for them in this iteration
- Services or systems NOT in the "Repos to modify" list are **reference context only** — do not write implementation steps, file changes, or instructions for them
- If the story requires coordinated changes across services, note cross-repo dependencies as: `[Cross-repo dependency: {service-name} — covered in that repo's spec]` for repos in scope, or `[Cross-service dependency: {service-name} — out of scope for this spec]` for repos not in scope

**Story Analysis:**
- Read the complete story thoroughly
- Identify business goals and user needs
- Extract acceptance criteria relevant to this repo (per repo tags from Phase 3)
- Assess technical feasibility and complexity
- Identify ambiguities requiring clarification

**Codebase Investigation (this repo only):**
- Use Grep/Glob to find relevant code files in the repo currently being specced
- Identify existing patterns and conventions
- Locate similar features for reference
- Understand current architecture and integration points
- Document specific files, functions, and line numbers

**Required Output:**
- Minimum 3-5 relevant file references with explanations (all from the repo currently being specced)
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
- Service name is the basename of the repo currently being specced (`git -C {repo_root} rev-parse --show-toplevel | xargs basename`)
- The notes adapter resolves `repo_root` to the repo currently being specced
- Follow notes adapter instructions to write the spec to the correct location within that repo

Record the spec path for use in the approval gate and Phase 11. When in the per-repo loop, do NOT confirm to the user after each individual spec — accumulate all paths and present them together at the approval gate.

---

## User Approval Gate

> **This gate runs after all per-repo specs are written (after the Phase 5–10 loop completes) and BEFORE Phase 11.** Do not proceed to Phase 11 without passing this gate.

**Interactive mode:**

Present all generated specs to the user in a summary table:

| Repo | Spec Path |
|------|-----------|
| [repo-name] | [path/to/spec.html] |
| … | … |

Ask:
> "The above specs have been written. Please review them and let me know:
> 1. Approve all — proceed to link specs and mark the story ready for development
> 2. Request changes to specific specs — describe what to revise"

- If the user **approves**: proceed to Phase 11.
- If the user **requests changes**: revise only the affected specs (re-run Phases 5–10 for those repos), re-present the updated specs, and repeat this gate. Do not proceed to Phase 11 until the user explicitly approves.

**Autonomous mode (cannot ask):**

Record all spec paths as written. Output a summary noting that user approval was skipped due to non-interactive execution, and list each repo and spec path. Proceed directly to Phase 11.

**Single-repo shortcut:** When only one repo is in scope, the approval gate still applies — present the single spec path and ask for approval before linking.

---

## Phase 11: Link Specs to PM Ticket

After user approval, update the **single original PM story** (not per-repo) to reference all generated specs.

Use the PM adapter to add a comment or description update to the story. The comment should include one entry per repo spec:
- The spec file path for each repo (relative to that repo's root)
- A brief summary of what each spec covers
- Example format (multi-repo):
  > **Implementation Specs Written**
  > - `[api]` Spec: `docs/specs/api/{story-id}.html` — Covers: [1-2 sentence summary]
  > - `[web]` Spec: `docs/specs/web/{story-id}.html` — Covers: [1-2 sentence summary]
  > Written by: Claude Write Spec workflow

If the PM adapter supports attaching files or adding external links, prefer adding one external link per repo spec (not per story). If it supports only one external link, add a comment instead with all paths.

If the PM adapter does not support comments or updates — note this to the user and provide all spec paths so they can link them manually.

**"Ready for Dev" transition and `claude-written` label:** Fire these **ONCE** on the single story after all specs are linked. State transitions and labels are applied once per run, not per repo.

---

## Phase 12: Adversarial Review

Read and follow the adversarial review procedure in `skills/shared/adversarial-review.md` with these context variables:

- `story_id`: the story ID from `$ARGUMENTS`
- `review_target`: "spec document"
- `review_context`: the full Claude Instructions spec content written in Phase 10

The adversarial agent verifies the spec completely and accurately captures all story requirements, acceptance criteria, and testing instructions. It checks that no story requirements were dropped, watered down, or misinterpreted.

---

