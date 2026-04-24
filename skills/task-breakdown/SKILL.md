---
name: task-breakdown
description: "Use this skill when the user already has a completed design or spec and wants to break it into tasks. The key signal: the design/spec already exists — the user is past the clarification phase. Trigger when the user says 'break this into tasks', 'create tickets', 'split this into work items', or provides a design document and asks for implementation planning. Also trigger when the user shares a spec/design from a previous conversation and says 'now break this down'. Do NOT trigger when there is no existing design — that's the workflow skill."
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

Each task follows this structure. Every field is required — a task missing any of them is not a standalone work order and will produce vague or incomplete code when picked up cold.

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

**Verification:**
- [ ] All tests pass
- [ ] [Specific check relevant to this task]
```

### Task Rules

**Self-contained.** A background agent with zero context reads this task and delivers working code. Include the why, not just the what. Include file paths. Include the spec. Include the commit points.

**Max 10 main files per task.** If a task touches more than 10 production files (test files don't count), split it. Large tasks are fragile — they create merge conflicts and are hard to review.

**Incremental commits.** Each TDD cycle = one commit. If something breaks mid-task, you lose minimal work. List commit points explicitly in the implementation plan.

**Verify at each step.** Run full test suite after each commit. Don't batch verification to the end.

**Follow TDD skill.** Every implementation step uses the TDD skill's RED-GREEN-REFACTOR cycle. Don't reinvent it here — reference it. The implementation plan lists *what* to test and implement; the TDD skill defines *how*.

**Implementation order.** Number tasks in the order they should be executed. Sequential tasks get consecutive numbers on the same track. Parallel tracks can start simultaneously.

### Self-check before presenting tasks

Before presenting the task list to the user, silently verify every task has all required fields:
- Track, Blocked by, Blocks
- Goal (one sentence)
- Constraints
- Spec (at least one GIVEN/WHEN/THEN)
- Files to touch (with descriptions)
- Implementation plan (numbered steps, each with a commit message)
- Verification checklist

If any field is missing, complete it before presenting. Never present an informal summary in place of the full template.

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
