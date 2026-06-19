---
name: dev-workflow:epic
description: "Use when a large initiative must be decomposed into many coordinated tasks and driven to completion, when a user says 'epic', 'break this down', 'orchestrate this feature', 'run this whole initiative', or '/start epic', or provides a high-level multi-task feature summary that spans more than one PR or repo. Resumable: re-invoke against an existing epic slug to continue an interrupted run."
---

# Epic

**Role:** Orchestrator — take one high-level feature summary, reach consensus on a full task
breakdown, then autonomously drive every task to a merged PR.

The orchestrator inherits the main session model — it must stay an orchestrator (see the mandate
below). Every unit of real work (brainstorming, repo discovery, spec writing, development, review,
testing, fixing) is dispatched to a **subagent**, whose model is resolved from config per
`skills/shared/standards.md` → "Subagent Model Selection".

## Orchestration-Only Mandate

**The orchestrator performs orchestration only.** Its sole direct actions are:

- the discovery interview and the single consensus gate (user-facing),
- writing and updating `tasklist.md` (via the `tasklist` adapter),
- computing the ready set and scheduling,
- dispatching subagents,
- merging an approved PR (via the git/gh wrapper) and marking the task done,
- the end-of-run report.

It **never** writes feature code, specs, reviews, or tests itself, and **never** creates a task
on its own initiative outside the consensus gate or a subagent's bug report. If you find yourself
authoring a spec, editing source, or reviewing a diff in the main agent — stop; that work belongs in
a dispatched subagent.

## Arguments: $ARGUMENTS

One of:

- **A high-level feature summary** — begin a new epic at Phase 1 (discovery).
- **An existing epic slug** (matches a directory under `~/.claude/dev-workflow/epics/`) — resume:
  jump to Phase 6 (scheduler), reading the existing `tasklist.md` as the source of truth.
- **Empty** — ask the user for the feature summary (interactive), or, in autonomous mode, stop and
  report that a summary is required.

Read `skills/shared/standards.md` — mandatory rules for this entire session.

Read `skills/shared/adapter-loading.md` — adapter loading procedure.

Read `skills/shared/repo-discovery.md` — repo discovery procedure used in Phase 1.

Read `skills/pm-adapter/tasklist.md` — the adapter that backs the tasklist; you operate it directly.

Read `skills/full-cycle/SKILL.md` — the per-task engine you drive; do not reimplement its loops.

Read the CLAUDE.md file in this repository before starting.

---

## Reuse, Do Not Reimplement

The per-task spec → development → review-loop → test-loop, resume/entry detection, and the loop-safety
guard all live in `full-cycle` already. **Epic drives `full-cycle` per task — it does not re-create
that logic.** The only net-new execution logic here is discovery, tasklist generation, and the
cross-task scheduler.

---

## Phase 0: Setup & Mode Detection

1. Determine output mode per `skills/shared/standards.md` → "Output Mode Detection". The discovery
   gate (Phase 5) requires user interaction; if running autonomously with no way to ask **and** this
   is a new epic (no existing slug), STOP and report that the consensus gate needs an interactive
   session. A **resume** against an existing slug needs no interaction and may run fully autonomously.
2. Read `~/.claude/dev-workflow/config.json`; note the `models` section for subagent dispatch.
3. The PM adapter for this skill is **always `tasklist`** (read `skills/pm-adapter/tasklist.md`) — do
   **not** use the configured `pm_adapter`, and do **not** rewrite `config.json`. The notes adapter
   is the configured one (used by `write-spec` inside each task's `full-cycle`).
4. If `$ARGUMENTS` is an existing epic slug → go to **Phase 6** (resume).

---

## Phase 1: Deep Discovery

Goal: understand the initiative well enough to propose a complete, correctly-ordered task breakdown.

1. **Brainstorm the initiative.** Dispatch a subagent (task type `reasoning`) instructed to invoke
   `superpowers:brainstorming` against the feature summary and return the clarified intent,
   requirements, edge cases, and risks. (Or, in interactive mode, run brainstorming in the main
   agent so it can ask the user — brainstorming is a discovery activity, not implementation work, so
   this does not violate the orchestration mandate.)
2. **Investigate the target repos.** Per `skills/shared/repo-discovery.md`, discover the repos in the
   workspace and their purposes. Dispatch repo-investigation to a subagent (task type `reasoning`)
   when the codebase reading is substantial, so raw file output stays out of the orchestrator context.
3. **Propose a decomposition** into small tasks. For each proposed task, mirror `create-story`'s
   field structure: `title`, `description`, `acceptanceCriteria`, `testingInstructions`, `repo`
   (exactly one per task), `story_type`, and `depends_on` (task IDs). A dependency exists when one
   task consumes an artifact another task produces (endpoint, contract, package, schema, file).
4. **Order and group.** Topologically sort tasks. Tasks in distinct repos with no dependency between
   them may run concurrently; tasks in the same repo are serialized (one in-flight PR per repo).

Do **not** write `tasklist.md` yet — nothing is created before the gate.

---

## Phase 5: Consensus Gate (single point of full agreement)

Present the **full proposed tasklist** to the user — the Mermaid ordering plus every task's
description, AC, testing, repo, and dependencies — and iterate until they **explicitly approve**.

- This is the **one** consensus gate. After approval the run is autonomous, including merges.
- Render the proposal per `skills/shared/standards.md` Output Format rules (HTML if written to a file
  for the user to read; otherwise the Mermaid + task list inline).
- In autonomous mode there is no gate to run — a new epic cannot reach this phase autonomously
  (Phase 0 stopped it). A resume skips this phase entirely.

Only on explicit approval, proceed to Phase 5b.

---

## Phase 5b: Write the Tasklist

Derive `[epic-slug]` from the feature summary (kebab-case, short). Create the directory
`~/.claude/dev-workflow/epics/[epic-slug]/`.

Write `tasklist.md` using the `tasklist` adapter's **Create story** capability for **each** approved
task — one Create call per task, in dependency order. The metadata header and Mermaid graph are
written per the file format in `skills/pm-adapter/tasklist.md`.

The Story Creation Gate is satisfied: the user explicitly invoked `epic` and approved this exact
task set. All creation happens here, in the orchestrator.

---

## Phase 6: Scheduler

The scheduler decides what runs next. It reads `tasklist.md` as the source of truth.

1. **Compute the ready set.** A task is *ready* when its status is `pending` and every task in its
   `depends_on` is `done`.
2. **Apply the parallelism rule:**
   - **One in-flight task per repo.** A repo is "in flight" if any of its tasks is `in-progress`,
     `in-review`, or `in-test`. Do not schedule a second task for a repo that already has one in
     flight.
   - **Cross-repo concurrency.** Ready tasks in distinct repos run **concurrently**, each as its own
     dispatched subagent.
   - **No git worktrees.** Each repo's single in-flight task works in that repo's own checkout. Do
     not invoke `superpowers:using-git-worktrees`; pass that override into every dispatch.
3. **Bug priority.** When an open bug task exists for a repo, schedule it as the **next** task for
   that repo — ahead of any pending feature task — subject to the same one-in-flight-PR-per-repo rule.
4. For every task selected this round, go to **Phase 7** (drive). Dispatch the concurrent ones in a
   single batch.
5. When the ready set is empty:
   - If any task is still in flight, wait for it to complete (Phase 8), then recompute from step 1.
   - If nothing is in flight and nothing is ready → go to **Phase 10** (terminate).

---

## Phase 7: Per-Task Drive

For each scheduled task, dispatch **one** subagent that runs `full-cycle` for that task in
**autonomous mode**, pinned to the `tasklist` adapter. Resolve the subagent model per
`standards.md`. The dispatch prompt must contain, explicitly:

> Invoke Skill: `dev-workflow:full-cycle` with task ID `{task_id}`, running autonomously.
>
> **PM adapter override:** Use the `tasklist` PM adapter (the plugin's built-in
> `skills/pm-adapter/tasklist.md`). The PM "story" is task `{task_id}` in the tasklist file
> `{tasklist_path}`. Do **not** read `pm_adapter` from config; do **not** contact
> Shortcut/Jira/Linear/GitHub Issues. This instruction overrides the configured adapter.
> Branch name: `{epic-slug}-{task_id}`. The PR carries **no** `sc-` ID.
>
> **Do NOT invoke `superpowers:using-git-worktrees`.** Work in this repo's own checkout. Pass this
> override into any nested `finishing-a-development-branch`.
>
> full-cycle enters at write-spec (the task exists with no spec) and proceeds through development,
> the review loop, and the test loop autonomously. **Do not merge** — report the PR's final review
> and test decisions back; the orchestrator merges.
>
> **Bug reporting:** If you discover a defect attributable to a previously-completed task, include in
> your result a `bug-report` describing the defect, the affected repo, and the suspected source task
> ID. **Do not create a task** — only report it.

The subagent returns its autonomous-mode key/value result. The orchestrator does **not** trust a
self-reported pass/fail — it re-confirms PR state authoritatively from GitHub (per full-cycle's
"Reading the Authoritative Review Decision") before merging.

---

## Phase 8: Completion + Merge

When a task's `full-cycle` run reports review-approved **and** test-approved:

1. Re-read the PR's authoritative review and test decisions from GitHub (dispatch the decision-read
   as a subagent so raw JSON stays out of the orchestrator). Both must be `APPROVED`.
2. **Merge** the PR using the git/gh wrapper (`gh-as-app.sh <persona> pr merge {PR} …`). This
   intentionally overrides full-cycle's "never merge" rule — for epic, merge-on-dual-approval is the
   completion step.
3. Via the `tasklist` adapter's **Update story**, set the task `Status` to `done` (which also updates
   its Mermaid node).
4. **If the merge cannot complete** (e.g. permissions, conflicts): set the task `Status` to `blocked`,
   record the reason as a comment via the adapter, and continue with other tasks — do **not** silently
   hang. *[Inference — explicit failure handling per the spec.]*
5. Recompute the ready set (Phase 6); newly unblocked tasks become schedulable.

---

## Phase 9: Bug Intake

When a task subagent's result contains a `bug-report`:

1. The **orchestrator** creates a new task via the `tasklist` adapter's **Create story** capability —
   `story_type: bug`, the reported `repo`, the reported description, and `depends_on` including the
   suspected **source task** (so the link to the originating task is recorded).
2. The bug task is scheduled with **priority** per Phase 6 step 3 — next for its repo, ahead of
   pending features — subject to one-in-flight-PR-per-repo.
3. When picked up, it flows through `full-cycle` identically to any other task, **including getting
   its own spec at write-spec time**.

The orchestrator creates the bug task; the subagent only reported it. This preserves "subagents do
work, they don't create tasks" and keeps creation inside the orchestrator's consensus authority.

---

## Phase 10: Blocked-Task Handling & Termination

**Blocked tasks.** A task is `blocked` when it exhausts full-cycle's loop-safety guard (review/test
never converges), cannot be merged, or otherwise cannot complete autonomously. Mark it `blocked` via
the adapter with the outstanding feedback recorded as a comment, then **continue with other unblocked
tasks**. A block on one task never halts the whole epic. *[Inference — derived from the spec's
error-scenario requirements.]*

**Termination.** Stop when no task is runnable — all tasks are `done`, or every remaining task is
`blocked`. Emit an end-of-run report:

- epic slug
- per-task final status
- PR URLs (and which tasks were merged)
- any blocked tasks, each with its reason

In autonomous mode, emit the flat key/value summary per `standards.md` (`service-name`, `pm-key`,
`pr-number`, `status`, `message`, plus highest-signal extras such as `branch`). For an epic the
`pm-key` is the epic slug and `pr-number` may be a count or the list of merged PRs.

---

## Resumability

`tasklist.md` **is** the durable epic state. On re-invocation against an existing slug:

1. Read `tasklist.md`; treat every task `Status` as authoritative.
2. Recompute the ready set from statuses (Phase 6). Do **not** recreate `done` tasks or re-merge
   merged PRs.
3. Resume scheduling. full-cycle's own per-task checkpoints under `~/.claude/dev-workflow/state/`
   remain in place for in-task resume — the tasklist is the cross-task source of truth, those
   checkpoints are the within-task one.

---

## Completion Criteria

- A new epic ran deep discovery and reached an explicit user-consensus gate before creating anything.
- `tasklist.md` was written at the configured path: a Mermaid graph ordering tasks with
  parallelization, each task's full description embedded.
- Each task was driven through `full-cycle` on the `tasklist` adapter with zero external PM calls;
  on dual approval its PR was merged and the task marked `done`.
- Parallelism held: cross-repo concurrent, same-repo sequential, one in-flight PR per repo, no
  worktrees.
- The orchestrator only orchestrated — no feature code, spec, review, or test authored in the main
  agent.
- Bug reports from subagents became orchestrator-created, priority-scheduled bug tasks.
- The run is resumable from `tasklist.md`, and a clear end-of-run report was produced.
