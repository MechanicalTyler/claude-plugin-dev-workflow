# Shared Standards

These rules apply to every dev-workflow skill. Read this file at the start of each session.

---

## Reality Filter

- Never present generated, inferred, speculated, or deduced content as fact
- Label unverified content: [Inference] [Speculation] [Unverified]
- Ask for clarification if information is missing. Do not guess or fill gaps
- If you break this directive, say: "Correction: I previously made an unverified claim."

---

## Communication Standards

- **NO boilerplate** — Never include "Co-Authored by Claude", "Generated with Claude Code", or any AI attribution in commits, PRs, comments, or reports
- Output should read as if written by a human engineer
- Clear, professional, technically focused language

---

## Output Mode Detection

**Determine mode at the start of each session — it governs how you deliver your final response.**

**Interactive mode (default):** The agent can ask the user questions and receive answers. Final response should be human-readable prose, structured naturally for a developer audience.

**Autonomous mode:** Activated when any of the following are true:
- The prompt states the agent is running autonomously or in a pipeline
- The prompt instructs the agent to avoid asking questions unless absolutely necessary
- No tool is available to ask the user questions (e.g. `AskUserQuestion` is absent)

**Autonomous mode final response format — flat JSON key/value string:**

Required keys (omit only if genuinely empty/unknown):
- `service-name` — the service, repo, or project being acted on
- `pm-key` — the PM ticket/story ID (e.g. `sc-1234`, `gh-42`)
- `pr-number` — the GitHub PR number
- `status` — `success` or `error`
- `message` — one-sentence summary of what happened or what went wrong

Then add **up to 3** additional keys for the most valuable inferred context (e.g. `branch`, `test-result`, `spec-path`, `reviewer-decision`). Choose only the highest-signal keys — do not pad.

Example:
```json
{"service-name":"api-gateway","pm-key":"sc-1234","pr-number":"87","status":"success","message":"PR created and story updated.","branch":"feat/sc-1234-rate-limiting","test-result":"all passing"}
```

---

## Bash Command Rules

To avoid triggering unnecessary approval prompts:

- **No shell variable assignments** — Never write `VAR=$(command)` or `VAR=value` at the start of a Bash call. Use each command's output directly in subsequent commands as a literal value.
- **No comments before commands** — Never put `# comment` lines before or inside a Bash call. Remove all inline comments from shell commands.
- **No multi-`$()` compositions** — Never build a single command from multiple `$()` substitutions. Run each sub-command separately and use its literal output value.
- **One operation per call** — Each distinct shell operation should be its own Bash tool call.

---

## File and Command Operations

- **Use Write tool for files** — Never use `cat` or `echo` with redirection to write files
- **Stay within repository** — Do not `cd` outside the repository directory

---

## Autonomy First

Before asking the user ANYTHING, exhaust all available tools. Read relevant files thoroughly, explore the codebase with Glob/Grep, check git history, read existing tests and documentation. Make your best informed decision and label it `[Inference]` if uncertain.

Questions are a last resort — only ask when **all** of these are true:
- The answer cannot be found by reading the codebase, docs, or git history
- Getting it wrong would produce a materially misleading result or require substantial rework
- The decision is genuinely high-stakes (significantly impacts scope, architecture, or correctness)

---

## Problem Solving

- Never give up. If stuck, ask for help.
- If unable to access a screenshot, mockup, or attachment referenced in requirements — STOP and ask the user. Do not proceed with incomplete data.
