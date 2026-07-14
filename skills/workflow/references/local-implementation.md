# Local Implementation — Task Execution Loop

The Task Execution Loop executes the confirmed task list locally — one task at a time. No re-analysis. The tasks are already designed; the only job here is to implement them correctly.

## Autonomy Contract

There is exactly one gate in this loop: asking the user for a branch name during Setup. Once the branch exists, run the entire task list to completion **without pausing for confirmation** — do not ask "shall I continue?", do not wait for approval to commit, do not stop between tasks. The tasks and design are already confirmed; stopping to check in would only stall a plan the user already approved.

Stop early only for a genuine blocker that makes correct progress impossible:
- The current task's description is ambiguous or contradicts the code, and no reasonable interpretation is safe.
- A test cannot be made to pass without a decision the task doesn't cover (missing dependency, external credential, a design gap).
- An action would be destructive or irreversible beyond the task's scope.

When you stop, say which task you stopped on and exactly what you need. Everything else — normal test failures, refactors, choosing between equivalent implementations — is yours to resolve inside the loop without asking.

## Setup

1. **Create a branch** — Ask the user for a branch name, or derive one from the feature slug. Create and check it out:
   ```
   git checkout -b <branch-name>
   ```

2. **Create `tasks.json`** — Convert `tasks.md` into `docs/<feature-slug>/tasks.json`:
   ```json
   {
     "feature": "<feature-slug>",
     "tasks": [
       {
         "id": "T1",
         "title": "Short task title",
         "description": "What this task implements",
         "depends_on": [],
         "status": "pending"
       }
     ]
   }
   ```
   Status values: `pending` | `in_progress` | `done` | `skipped`

Execute in dependency order — a task is eligible only when all its `depends_on` are `done`.

## Per-Task Loop

Run this loop unattended, task after task, until all tasks are `done` or a genuine blocker (see Autonomy Contract) forces a stop. For each eligible task:

1. **Read** — Load the task. Mark it `in_progress` in `tasks.json`.

2. **Implement** — Follow the **tdd** skill (Red-Green-Refactor):
   - Write a failing test first (RED)
   - Write minimum code to make it pass (GREEN)
   - Refactor while green (REFACTOR)

3. **Verify** — Run the full test suite. All tests must pass before moving on.

4. **Commit** — Commit with a message scoped to this task. Append a changelog entry.

5. **Mark done** — Update `status` to `done` in `tasks.json`. Pick the next eligible task.

6. **Clear context** — Once a task is committed and marked `done`, its implementation detail (file reads, test output, diffs) has no further use — everything that matters is now in git and in `tasks.json`. Clear/compact context before starting the next task so it doesn't accumulate across the run. This is what keeps a long task list from overloading a single context: each task starts lean, reading only `tasks.json` and the current code. Clearing context is part of the loop, not an interruption of it — do it automatically, without asking.

Repeat until all tasks are `done`.

## Resuming the Loop

Whether resuming after a deliberate context clear or an unplanned reset (session restart, auto-compaction):

1. Read `tasks.json` — find the first task that is `in_progress` (an interrupted task) or the first `pending` task whose `depends_on` are all `done`.
2. Read that task's description. Do not rely on memory of prior tasks — the task description and the current code are the only inputs you need.
3. Check `git log` / `git status` to confirm what's actually committed vs. what an interrupted task left half-done. If a task is `in_progress` but there's no matching commit, treat it as not yet started and redo it from RED.
4. Resume the Per-Task Loop from step 2 (Implement).

## Completion

When all tasks are `done`:
- Summarize: tasks completed, any skipped and why, final test status
- Update the manifest: all stages `confirmed`