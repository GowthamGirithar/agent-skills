# Local Implementation — Task Execution Loop

The Task Execution Loop executes the confirmed task list locally — one task at a time. No re-analysis. The tasks are already designed; the only job here is to implement them correctly.

Each task becomes its own **stacked branch and pull request**. This is the point of the whole pipeline: `task-breakdown` produced self-contained, dependency-ordered tasks so that a reviewer gets small, focused PRs instead of one monolith. If every task landed on a single branch, that work would be wasted at the last step. Stacking keeps each PR small while respecting the dependency graph — a dependent task branches off its dependency's branch, so it can build and pass tests against real code without waiting for anything to merge.

## Autonomy Contract

The Setup gate is the one place the loop stops for the user. It confirms the base branch, the branch-naming scheme, and that pushing + opening PRs is possible (remote reachable, `gh` authenticated). Confirming Setup authorizes the loop to push branches and open PRs autonomously for the rest of the run — that is why the outward-facing steps don't re-prompt on every task.

Once Setup passes, run the entire task list to completion **without pausing for confirmation** — do not ask "shall I continue?", do not wait for approval to commit, push, or open a PR, do not stop between tasks. The tasks and design are already confirmed; stopping to check in would only stall a plan the user already approved.

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

4. **Prepare `tasks.json`** — Convert `tasks.md` into `docs/<feature-slug>/tasks.json`. Each task carries a `branch` and `pr_url` field (initially `null`) so the loop can compute parent bases and resume cleanly:
   ```json
   {
     "feature": "<feature-slug>",
     "base_branch": "main",
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

## Per-Task Loop

Run this loop unattended, task after task, until all tasks are `done` or a genuine blocker (see Autonomy Contract) forces a stop. For each eligible task:

1. **Read** — Load the task. Mark it `in_progress` in `tasks.json`.

2. **Cut the branch** — Where the branch is cut from depends on the task's dependencies. This is what makes the stack dependency-safe:

   | `depends_on`        | Cut the branch from            | PR base            |
   |---------------------|--------------------------------|--------------------|
   | `[]` (none)         | the base branch                | the base branch    |
   | `[X]` (single)      | X's branch                     | X's branch         |
   | `[X, Y, …]` (multi) | the primary dep's branch, then `git merge` the other dep branches into it | the primary dep's branch |

   The "primary" dependency for a multi-dep task is whichever is furthest downstream (the longest existing stack); merging the others in gives the task all the code it needs. Record the branch name in the task's `branch` field.
   ```
   git checkout <parent-branch>        # base branch, or the dependency's branch
   git checkout -b <feature-slug>/<task-id>-<task-slug>
   # for a multi-dep task, also: git merge <other-dep-branch> ...
   ```

3. **Implement** — Follow the **tdd** skill (Red-Green-Refactor):
   - Write a failing test first (RED)
   - Write minimum code to make it pass (GREEN)
   - Refactor while green (REFACTOR)

4. **Verify** — Run the full test suite. All tests must pass before moving on.

5. **Commit** — Commit with a message scoped to this task, using Conventional Commits format: `<type>(<scope>): <description>`. Pick the `type` from the table below. Append a changelog entry.

   | Type       | Purpose                                 |
   |------------|-----------------------------------------|
   | `feat`     | New feature                             |
   | `fix`      | Bug fix                                 |
   | `docs`     | Documentation only                      |
   | `style`    | Formatting/style (no logic)             |
   | `refactor` | Code refactor (no feature/fix)          |
   | `perf`     | Performance improvement                 |
   | `test`     | Add/update tests                        |
   | `build`    | Build system/dependencies               |
   | `ci`       | CI/config changes                       |
   | `chore`    | Maintenance/misc                        |
   | `revert`   | Revert commit                           |

6. **Push and open the PR** — Push the branch and open a PR whose base is the parent branch from step 2 (the base branch for an independent task, the dependency's branch for a stacked one):
   ```
   git push -u origin <branch>
   gh pr create --base <parent-branch> --head <branch> --title "<type>(<scope>): <title>" --body "<body>"
   ```
   Use this PR body so the reviewer understands where the PR sits in the stack:
   ```markdown
   ## Task <task-id>: <title>

   <task description>

   **Depends on:** <task-ids, or "none">
   **Stacked on:** #<parent-PR-number> (or "base branch — merge independently")

   > Part of a stacked PR set for `<feature-slug>`. Merge bottom-up.
   ```
   Record the returned URL in the task's `pr_url` field.

7. **Mark done** — Update `status` to `done` in `tasks.json`. Pick the next eligible task.

8. **Clear context** — Once a task is committed, pushed, PR'd, and marked `done`, its implementation detail (file reads, test output, diffs) has no further use — everything that matters is now in git, in the PR, and in `tasks.json`. Clear/compact context before starting the next task so it doesn't accumulate across the run. This is what keeps a long task list from overloading a single context: each task starts lean, reading only `tasks.json` and the current code. Clearing context is part of the loop, not an interruption of it — do it automatically, without asking.

Repeat until all tasks are `done`.

## Resuming the Loop

Whether resuming after a deliberate context clear or an unplanned reset (session restart, auto-compaction), reconcile intent (`tasks.json`) against reality (git + open PRs) — don't rely on memory:

1. Read `tasks.json` — find the first task that is `in_progress` (an interrupted task) or the first `pending` task whose `depends_on` are all `done`.
2. Read that task's description. The task description and the current code are the only inputs you need.
3. Reconcile what actually exists:
   - `git branch --list` and `gh pr list` show which branches/PRs are real, independent of what `tasks.json` claims.
   - An `in_progress` task with a branch **and** an open PR → treat as `done`; update `tasks.json` and move on.
   - An `in_progress` task with a branch and commits but **no PR** → finish verification, push, and open the PR (step 6), then mark `done`.
   - An `in_progress` task with a branch but no matching commit → treat as not yet started; delete the stray branch and redo from RED.
4. Resume the Per-Task Loop from step 2 (Cut the branch) or step 3 (Implement), depending on what already exists.

## Completion

When all tasks are `done`:
- Summarize: tasks completed, any skipped and why, final test status.
- List every PR with its `pr_url`, grouped into the **bottom-up merge order** — dependencies before dependents. The reviewer merges the stack from the bottom up.
- **Flag the stacked-merge caveat:** squash-merging a lower PR rewrites the commits its children stacked on, so each child needs a rebase onto the updated base afterward. To avoid churn, either merge the whole stack bottom-up in one sitting, or use stacked-PR tooling (e.g. Graphite, `spr`) if the team has it. Say this explicitly so no one is surprised when child PRs show conflicts after the parent merges.
- Update the manifest: all stages `confirmed`.
