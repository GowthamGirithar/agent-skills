# Local Implementation — Task Execution Loop

The Task Execution Loop executes the confirmed task list locally — one task at a time. No re-analysis. The tasks are already designed; the only job here is to implement them correctly.

Each task becomes its own **stacked branch and pull request**. This is the point of the whole pipeline: `task-breakdown` produced self-contained, dependency-ordered tasks so that a reviewer gets small, focused PRs instead of one monolith. If every task landed on a single branch, that work would be wasted at the last step. Stacking keeps each PR small while respecting the dependency graph — a dependent task branches off its dependency's branch, so it can build and pass tests against real code without waiting for anything to merge.

## Autonomy Contract

The Setup gate is the one place the loop stops for the user. It confirms the base branch, the branch-naming scheme, and that pushing + opening PRs is possible (remote reachable, `gh` authenticated). Confirming Setup authorizes the loop to push branches and open PRs autonomously for the rest of the run — that is why the outward-facing steps don't re-prompt on every task.

Once Setup passes, run the entire task list to completion **without pausing for confirmation** — do not ask "shall I continue?", do not wait for approval to commit, push, or open a PR, do not stop between tasks. The tasks and design are already confirmed; stopping to check in would only stall a plan the user already approved.

Note: passing Setup stops the *loop* from pausing between tasks — it can't suppress the harness's own per-tool permission prompts, which depend on the user's permission mode. Mention this at Setup so the user can switch modes if they want zero interruptions.

Delegation applies to every task list, no matter how short — never implement inline instead of spawning a sub-agent. Before any Edit/Write in the orchestrator's own context, ask: *am I delegating this, or doing it myself?*

Stop early only for a genuine blocker that makes correct progress impossible:
- The current task's description is ambiguous or contradicts the code, and no reasonable interpretation is safe.
- A test cannot be made to pass without a decision the task doesn't cover (missing dependency, external credential, a design gap).
- `gh` is not installed/authenticated, the remote is unreachable, or a `push`/PR creation is rejected — the loop can't produce the PRs it's supposed to.
- An action would be destructive or irreversible beyond the task's scope.

When you stop, say which task you stopped on and exactly what you need. Everything else — normal test failures, refactors, choosing between equivalent implementations — is yours to resolve inside the loop without asking.

## Setup

This is the one gate. Settle all of it before the first line of code:

1. **Confirm the base branch** — the branch the stack ultimately merges into (default `main`, or whatever the repo's default branch is). Independent tasks branch from here.

2. **Confirm the branch-naming scheme** — default `<feature-slug>/<task-id>-<task-slug>`, e.g. `cart-validation/t1-add-schema`. Consistent names make the stack legible in `git branch` and in the PR list.

3. **Confirm push + PR is possible** — the remote is reachable and `gh` is authenticated (`gh auth status`). If `gh` is missing or not authenticated, stop here — this is a blocker, because the chosen path can't deliver its PRs otherwise.

4. **Prepare `tasks.json`** — Convert `tasks.md` into `docs/<feature-slug>/tasks.json`. Each task carries a `branch` and `pr_url` field (initially `null`) so the loop can compute parent bases and resume cleanly. A top-level `notes` array holds cross-task learnings the orchestrator carries forward from one sub-agent to the next (see [Cross-Task Learning](#cross-task-learning)):
   ```json
   {
     "feature": "<feature-slug>",
     "base_branch": "main",
     "notes": [],
     "tasks": [
       {
         "id": "T1",
         "title": "Short task title",
         "description": "What this task implements",
         "depends_on": [],
         "status": "pending",
         "branch": null,
         "pr_url": null
       }
     ]
   }
   ```
   Status values: `pending` | `in_progress` | `done` | `skipped`

Execute in dependency order — a task is eligible only when all its `depends_on` are `done` (a `done` task has a branch and, normally, an open PR).

## The Loop: Orchestrator + One Sub-Agent Per Task

The loop runs as two roles. Keeping them separate is what keeps context lean across a long task list.

- **Orchestrator (this main session)** — holds the Setup gate, picks the next eligible task, spawns a sub-agent to implement it, records the result in `tasks.json`, carries learnings forward, and decides whether to continue or stop on a blocker. The orchestrator never reads task diffs, file contents, or test output itself — that all lives in the sub-agent's context and is discarded when the sub-agent returns. The orchestrator accumulates only compact summaries plus the design docs, so it scales across a long task list far better than one session implementing every task inline.
- **Per-task sub-agent** — a fresh, isolated context window that implements exactly one task end-to-end and returns a compact summary. Spawn it with the Agent tool (`subagent_type: general-purpose`).

**Why a sub-agent instead of "clear context between tasks":** the main session has no tool to clear or compact its own context mid-run — `/clear` and `/compact` are user-typed commands, and auto-compaction only fires when the window fills. Delegating each task to a sub-agent is the only mechanism that actually gives every task a lean, fresh window: the sub-agent's file reads, diffs, and test output vanish when it returns, and everything that matters is already in git, in the PR, and in `tasks.json`.

**Sequential, never parallel.** Stacked branches are dependency-ordered, and every sub-agent shares the same working directory and git repo. Spawn one sub-agent at a time (`run_in_background: false`) and wait for it to return before spawning the next. Parallel sub-agents would race on the branch stack and the working tree and corrupt both.

### Orchestrator loop

Run this unattended, task after task, until all tasks are `done` or a genuine blocker (see Autonomy Contract) forces a stop. For each eligible task (all its `depends_on` are `done`), in dependency order:

1. **Mark `in_progress`** — Set the task's `status` to `in_progress` in `tasks.json`.
2. **Compute the parent branch** — This is the sub-agent's `git` starting point, and the PR base:

   | `depends_on`        | Cut the branch from            | PR base            |
   |---------------------|--------------------------------|--------------------|
   | `[]` (none)         | the base branch                | the base branch    |
   | `[X]` (single)      | X's branch                     | X's branch         |
   | `[X, Y, …]` (multi) | the primary dep's branch, then `git merge` the other dep branches into it | the primary dep's branch |

   The "primary" dependency for a multi-dep task is whichever is furthest downstream (the longest existing stack); merging the others in gives the task all the code it needs. Read the parent branch name from the dependency's `branch` field in `tasks.json`.
3. **Spawn the sub-agent** — one task, `run_in_background: false`, using the prompt template below. Include the current `notes` array so the sub-agent inherits prior learnings.
4. **Wait for the summary** — do not spawn the next task until this one returns.
5. **Record the result** — write the sub-agent's `branch` and `pr_url` into the task, set `status` to `done`, and append any new learnings to the top-level `notes` array.
6. **Handle a blocker** — if the sub-agent returns `blocked` instead of `done`, stop and report per the Autonomy Contract (which task, which step, what's needed). Leave the task `in_progress` so the resume path can pick it up.
7. **Next task** — pick the next eligible task and repeat.

Repeat until all tasks are `done`.

### The sub-agent prompt

The sub-agent has no memory of prior tasks — give it everything it needs to run blind. Template:

```
Implement exactly one task from a stacked-PR feature. Follow the tdd skill.

Feature: <feature-slug>
Task <id>: <title>
Description: <full task description>
Depends on: <task-ids, or "none">

Parent branch (cut from this, and use as the PR base): <parent-branch>
New branch to create: <feature-slug>/<task-id>-<task-slug>
For a multi-dependency task, after cutting the branch also merge: <other-dep-branches>

Carry-forward notes from earlier tasks (may be empty):
<notes array — one line each>

Steps:
1. git checkout <parent-branch> && git checkout -b <new-branch>
   (multi-dep: also git merge <other-dep-branch> ...)
2. Implement with TDD (RED → GREEN → REFACTOR).
3. Run the full test suite — all tests must pass.
4. Commit with a Conventional Commits message: <type>(<scope>): <description>. Append a changelog entry.
5. git push -u origin <new-branch>
6. gh pr create --base <parent-branch> --head <new-branch> --title "<type>(<scope>): <title>" --body "<body>"
   (PR body template below)
7. Return the summary described below. Do NOT proceed to any other task.

PR body:
## Task <task-id>: <title>

<task description>

**Depends on:** <task-ids, or "none">
**Stacked on:** #<parent-PR-number> (or "base branch — merge independently")

> Part of a stacked PR set for `<feature-slug>`. Merge bottom-up.

Conventional Commit types: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert.

Return exactly this summary (nothing else):
- status: done | blocked
- branch: <branch name>
- pr_url: <url, or null if blocked>
- learnings: 0–3 short notes useful to LATER tasks (test-harness setup, where shared helpers live, a non-obvious gotcha), or "none"
- if blocked: which step, and exactly what decision or input is needed
```

### The summary the sub-agent returns

Keep it compact — this is all that re-enters the orchestrator's context:

- **status** — `done` or `blocked`
- **branch** — the branch it created
- **pr_url** — the opened PR (or `null` if blocked)
- **learnings** — 0–3 short notes for later tasks, or none
- **if blocked** — which step it stopped on and what's needed

### Cross-Task Learning

Tasks are self-contained, but implementation surfaces things later tasks benefit from — where test fixtures live, how the local DB is seeded, a build quirk, a shared helper's location. The orchestrator keeps these in the top-level `notes` array in `tasks.json` and passes them into every subsequent sub-agent's prompt, so each fresh sub-agent starts blind on *its* task but not on the codebase's quirks.

Keep `notes` short and cross-cutting — a handful of lines, not a running log. Good notes generalize ("integration tests need `make seed-db` first"; "auth mocks are in `test/support/auth.ex`"). Skip anything task-specific that's already captured in the code, the commit, or the PR — that detail belongs there, not in `notes`.

### When sub-agents aren't available

On surfaces without sub-agents (e.g. Claude.ai), the orchestrator/sub-agent split can't run. Fall back to implementing each task inline in the main session, following the same per-task steps, and rely on the harness's auto-compaction between tasks as a best-effort substitute. State this plainly to the user: context won't reset cleanly per task, so a long task list may hit compaction mid-run — the resume path below still recovers correctly because it reconciles against git and `tasks.json`, not memory.

## Resuming the Loop

The orchestrator resumes here after any reset — a sub-agent that died mid-task, a session restart, or auto-compaction. Reconcile intent (`tasks.json`) against reality (git + open PRs); don't rely on memory:

1. Read `tasks.json` — including the `notes` array (the carry-forward learnings to feed the next sub-agent) — and find the first task that is `in_progress` (an interrupted task) or the first `pending` task whose `depends_on` are all `done`.
2. Read that task's description. The task description, the `notes`, and the current code are the only inputs a fresh sub-agent needs.
3. Reconcile what actually exists:
   - `git branch --list` and `gh pr list` show which branches/PRs are real, independent of what `tasks.json` claims.
   - An `in_progress` task with a branch **and** an open PR → treat as `done`; update `tasks.json` and move on.
   - An `in_progress` task with a branch and commits but **no PR** → have a sub-agent finish verification, push, and open the PR, then mark `done`.
   - An `in_progress` task with a branch but no matching commit → treat as not yet started; delete the stray branch and re-spawn the sub-agent from RED.
4. Resume the orchestrator loop: spawn the next per-task sub-agent (step 3 of the Orchestrator loop), telling it where to pick up based on what already exists.

## Completion

When all tasks are `done`:
- Summarize: tasks completed, any skipped and why, final test status.
- List every PR with its `pr_url`, grouped into the **bottom-up merge order** — dependencies before dependents. The reviewer merges the stack from the bottom up.
- **Flag the stacked-merge caveat:** squash-merging a lower PR rewrites the commits its children stacked on, so each child needs a rebase onto the updated base afterward. To avoid churn, either merge the whole stack bottom-up in one sitting, or use stacked-PR tooling (e.g. Graphite, `spr`) if the team has it. Say this explicitly so no one is surprised when child PRs show conflicts after the parent merges.
- Update the manifest: all stages `confirmed`.
