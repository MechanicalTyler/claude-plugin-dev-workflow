---
name: story-starter
description: "Conduct an interactive interview (max 2 clarifying questions) to generate a structured PM story draft — with title, description, acceptance criteria, and testing instructions — then create the approved story in the configured PM tool (Shortcut, Linear, Jira, GitHub Issues). Use when /start story-starter is invoked, or whenever a user wants to capture a feature idea or request as a formal story. Also use when a user says 'create a story', 'write a ticket', 'add to backlog', or describes a feature they want tracked."
---

# Story Starter

**Role:** Story Starter — interview the user, generate a story draft, and submit it to the PM tool

## Arguments: $ARGUMENTS

No required arguments. Optional: a brief feature description as a starting prompt.

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

---

## Phase 1: Load Adapters

1. Read `~/.claude/dev-workflow/config.json`
2. Note the `pm_adapter` value
3. Load PM adapter: check `~/.claude/skills/pm-adapter/{pm_adapter}.md` first (user override); fall back to `skills/pm-adapter/{pm_adapter}.md`
4. If neither exists: STOP — "PM adapter '{name}' not found. Check ~/.claude/dev-workflow/config.json and ensure the adapter file exists."
5. Confirm the adapter implements **Create Story** (capability #5) by checking for a `## Create Story` heading in the loaded adapter file. If not found: STOP — "This PM adapter does not support Create Story. Please update ~/.claude/skills/pm-adapter/{name}.md with a Create Story section."
6. Check whether the loaded adapter has pre-flight requirements (e.g., Linear requires `teamId`, Jira requires `PROJECT_KEY` if not yet established in session). Surface any such requirements to the user **before** starting the interview in Phase 3, so they don't interrupt Phase 7.

---

## Phase 2: Investigate Repos

1. Determine the workspace root — use the current working directory
2. Find all repos in or below the workspace:
   - First check if CWD itself is a repo: look for `.git` at the workspace root
   - Then use Glob to find `{workspace}/*/.git` — each parent is a repo
   - Deduplicate results
3. For each repo found, read `CLAUDE.md` from the repo root:
   - If `CLAUDE.md` exists: use its content as the repo context
   - If `CLAUDE.md` is absent: note a warning — "⚠️ No CLAUDE.md found for {repo-name} — skipping context for this repo" — and continue
4. Compile repo briefs as structured summaries:
   ```
   **{repo-name}**: {one-sentence description from README or CLAUDE.md}
   Stack: {languages/frameworks from manifest}
   Key dirs: {top-level directories}
   ```
5. If no repos found: note this — Phase 3 will prompt the user for context

---

## Phase 3: Initial Prompt Collection

- If `$ARGUMENTS` is non-empty: use it as the initial feature description — skip asking
- If `$ARGUMENTS` is empty and workspace has repos: ask — "What feature would you like to build?"
- If `$ARGUMENTS` is empty and no repos found: ask — "What feature would you like to build? (No repos detected in workspace — please also describe the target repo)"

---

## Phase 4: Interview (Max 2 Questions)

Using the repo briefs and initial prompt, attempt to populate all story draft fields.

Field derivation rules:
- `title`: derive from the starter prompt — **never ask**
- `originalRequest`: verbatim user starter prompt — **never ask**
- `description`: summarize from starter prompt — ask only if the request is genuinely too vague to understand
- `repoToModify`: match to the most relevant repo brief — ask only if no repo clearly fits or there are no repos
- `reposToReference`: infer from repo briefs (repos that provide context without being modified) — **never ask**
- `acceptanceCriteria`: derive from repo patterns and feature description — **never ask** (infer up to 5 items)
- `testingInstructions`: derive from repo patterns — **never ask** (infer up to 3 steps)
- `story_type`: infer from context — "feature" for new capabilities, "bug" for fixes, "chore" for maintenance — **never ask**

Use the `AskUserQuestion` tool for any clarifying questions. **Stop asking after 2 questions maximum** — draft regardless of remaining ambiguity, labeling uncertain fields as [Inference].

**Note:** The initial "What feature would you like to build?" prompt in Phase 3 does **not** count against the 2-question limit. The limit applies only to clarifying questions asked during Phase 4.

---

## Phase 5: Generate Story Draft

Emit the draft as a fenced JSON block:

~~~
```story-draft
{
  "title": "[repo-name] brief title describing the feature",
  "description": "2-3 sentence description of what needs to be built and why",
  "originalRequest": "verbatim user starter prompt",
  "repoToModify": "repo-name",
  "reposToReference": ["other-repo-1", "other-repo-2"],
  "acceptanceCriteria": [
    "First acceptance criterion (max 5 total)",
    "Second acceptance criterion"
  ],
  "testingInstructions": [
    "Step 1: how to test (max 3 total)",
    "Step 2: verify expected behavior"
  ],
  "story_type": "feature"
}
```
~~~

Then display the draft in a readable format:

```
**Title:** {title}
**Description:** {description}
**Repo to modify:** {repoToModify}
**Repos to reference:** {reposToReference joined with ", " or "(none)"}

**Acceptance Criteria:**
- [ ] {ac item 1}
- [ ] {ac item 2}
...

**Testing Instructions:**
1. {step 1}
2. {step 2}
...
```

---

## Phase 6: Approval

Ask the user:

> "Type **yes** to submit this story to {pm_adapter}, or describe what to change."

- If user says "yes" (case-insensitive): proceed to Phase 7
- If user provides feedback: incorporate the feedback, return to Phase 5 and re-emit an updated draft. After 3 revision cycles without approval, suggest the user cancel and restart with a clearer description.
- If user says "stop", "quit", "cancel", or "exit": STOP — "Story creation cancelled."

---

## Phase 7: Create Story

Use the PM adapter's **Create Story** operation (capability #5) with all draft fields.

On success: display —
```
Story created successfully!
ID: {story-id}
URL: {story-url}
```

On failure: STOP with the error — do not retry silently.
