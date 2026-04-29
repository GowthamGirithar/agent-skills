---
name: task-breakdown
description: "Use when the user has a completed design or spec and wants it split into implementable tasks. Key signal: spec already exists, user is past clarification. Trigger: 'break into tasks', 'create tickets', 'split into work items'. Do NOT trigger without an existing design — use workflow instead."
---

# Task Breakdown

You are a principal architect splitting a design into implementable work units. Each task is a standalone work order — complete enough that a background agent picks it up cold and delivers working code without needing the original design document.

## Phase 1: Analyze the Design

Read the design output (from the requirement-design skill or whatever the user provides). Extract:

- All behavioral changes (from GIVEN/WHEN/THEN specs)
- Data model changes
- API surface changes
- Infrastructure/config changes
- Migration needs
- Documentation impact — identify READMEs, API docs, runbooks, ADRs, or config references that will need updating based on the changes above

Map each change to the files it touches. Use subagents (Explore type) to identify exact file paths and understand the current code structure.

## Phase 2: Build the Dependency Graph

For every change identified, determine:

1. **What depends on what.** Data model changes before API changes. API changes before UI changes. Shared utilities before consumers.
2. **What is independent.** Changes that touch different files with no shared state can run in parallel.
3. **What is sequential.** Changes where output of one is input to another. These form chains.

Draw the graph. Group independent chains into parallel tracks.

Present the dependency graph to the user. Format:

```
Track A (parallel):
  A1: [task name] → A2: [task name]

Track B (parallel):
  B1: [task name]

Track C (parallel, blocked by A2):
  C1: [task name] → C2: [task name]
```

## Phase 3: Define Tasks

Each task follows the template below. Every field is required — no exceptions. The reason: a background agent reads this task cold with zero context. If a field is missing, the agent guesses, and guesses produce bugs. These fields aren't boilerplate — they're the difference between a task that ships and a task that gets sent back.

### Task Template

```
### Task [ID]: [Name]

**Track:** [which parallel track]
**Blocked by:** [task IDs, or "none"]
**Blocks:** [task IDs, or "none"]

**Goal:** What this task achieves. One sentence.

**Constraints:**
- [Hard limits relevant to this task]

**Spec:**
GIVEN [precondition]
WHEN [action]
THEN [outcome]
(Include only the specs from the design that this task implements)

**Files to touch:**
- `path/to/file.go` — what changes here
- `path/to/file_test.go` — test for above

**Implementation plan:**
Follow the TDD skill for each step. List the steps here:
1. [Specific behavior to test and implement]
   Commit: `[commit message]`
2. [Next behavior]
   Commit: `[commit message]`
(Each step = one TDD cycle + one commit)

**Documentation:**
- `path/to/README.md` — update API usage examples
- `path/to/docs/runbook.md` — add new alert handling section
(List docs that need updating due to this task's changes. Include READMEs, API docs, runbooks, ADRs, or config references. Write "none" if no doc changes needed.)

**Verification:**
- [ ] All tests pass
- [ ] Documentation updated matches the code changes
- [ ] [Specific check relevant to this task]
```

### Why each field matters

These explanations exist so you understand the purpose of each field — not as optional commentary.

- **Track / Blocked by / Blocks:** Without these on each task, the agent or developer doesn't know what to wait for or what to start. The dependency graph alone isn't enough — each task must carry its own placement info.
- **Goal:** Forces clarity. If you can't say what a task achieves in one sentence, the task is too vague or too broad.
- **Constraints:** A background agent doesn't know the project's unwritten rules. Write them down per task — things like "must not break backward compatibility", "must use existing database connection pool", "proto dependency must be bumped first".
- **Spec (GIVEN/WHEN/THEN):** This is the most important field. The overall design has specs, but each task must carry the subset of specs it implements. A background agent won't read the design doc. It reads this task. If the spec isn't here, the agent implements something that compiles but doesn't match the requirement. Copy the relevant specs from the design — don't summarize, don't paraphrase, carry them over exactly.
- **Files to touch:** Not just filenames — describe what changes in each file. "get_pricing.go — add mapItemCharge function" tells the agent where to work and what to add.
- **Implementation plan with per-step commits:** Each step is one TDD cycle. One commit per step means if something breaks, you lose one step, not the whole task. List what to test and implement at each step, with the commit message.
- **Documentation:** Code changes often require doc updates — a new function needs API doc updates, a new config flag needs runbook entries, a behavioral change needs README updates. If you don't flag these, they get forgotten until someone discovers stale docs in production. Explicitly write "none" if no docs need updating so it's clear you considered it.
- **Verification:** A checklist the agent runs through before marking the task done. Include task-specific checks beyond "tests pass" — things like "new charge types appear in the proto import", "zero-value fees produce no charge entry".

### Task Rules

**Self-contained.** A background agent with zero context reads this task and delivers working code. Include the why, not just the what. Include file paths. Include the spec. Include the commit points.

**Max 10 main files per task.** If a task touches more than 10 production files (test files don't count), split it. Large tasks are fragile — they create merge conflicts and are hard to review.

**Incremental commits.** Each TDD cycle = one commit. If something breaks mid-task, you lose minimal work. List commit points explicitly in the implementation plan.

**Verify at each step.** Run full test suite after each commit. Don't batch verification to the end.

**Follow TDD skill.** Every implementation step uses the TDD skill's RED-GREEN-REFACTOR cycle. Don't reinvent it here — reference it. The implementation plan lists *what* to test and implement; the TDD skill defines *how*.

**Implementation order.** Number tasks in the order they should be executed. Sequential tasks get consecutive numbers on the same track. Parallel tracks can start simultaneously.

## Pre-presentation gate

Before presenting the task list to the user, stop and verify every task against this checklist. Do not present tasks until every box is checked for every task.

For each task, confirm:
- [ ] Has Track, Blocked by, Blocks
- [ ] Has Goal (one sentence, not a paragraph)
- [ ] Has Constraints (at least one, or explicitly "none")
- [ ] Has Spec with at least one GIVEN/WHEN/THEN (copied from the design, not summarized)
- [ ] Has Files to touch with descriptions of what changes (not just filenames)
- [ ] Has Implementation plan with numbered steps, each with a commit message
- [ ] Has Documentation listing docs to update or explicitly "none"
- [ ] Has Verification checklist with task-specific checks

If any field is missing on any task, fill it in before presenting. Never present an informal summary, a simplified format, or a "condensed" version in place of the full template. The template is the deliverable.

## Phase 4: Review with User

Present all tasks. Ask:
- Are the task boundaries right?
- Anything missing?
- Should any tasks be merged or split further?
- Any constraints I missed?

Adjust based on feedback.

## Phase 5: Jira Integration

After tasks are finalized, ask:

> "Create these as Jira subtasks? If yes, provide the parent ticket key (e.g., PROJ-123)."

If user says yes:

1. **Update parent ticket** with the full design and spec in the description (use Atlassian MCP `editJiraIssue`).

2. **Create subtasks** under the parent (use Atlassian MCP `createJiraIssue` with `parent` field). Each subtask description contains:
   - Goal
   - Constraints
   - Spec (GIVEN/WHEN/THEN)
   - Implementation plan (with commit points)
   - Files to touch
   - Documentation
   - Verification checklist

3. **Link dependencies** between subtasks that are sequential (use Atlassian MCP `createIssueLink` with "Blocks" link type). For tasks where A must complete before B:
   - inwardIssue = A (the blocker)
   - outwardIssue = B (the blocked)

4. Report back the created ticket keys and their dependency links.

If user says no, output the task breakdown as markdown. The breakdown is the deliverable either way — Jira is just the tracking layer.

## Principles

- Tasks are work orders, not wish lists. Vague tasks produce vague code.
- A task that needs "see the design doc" is incomplete. Inline everything the agent needs.
- Parallel tracks reduce calendar time. Find them aggressively.
- Sequential dependencies are real. Don't pretend things are parallel when they share state.
- 10-file limit is a smell test. If you need more, the task is doing too much.
- Commit early, commit often. Each TDD cycle = one commit.
- The template exists because shortcuts here create rework downstream. Every skipped field is a question a background agent has to guess at — and guesses are bugs waiting to happen.