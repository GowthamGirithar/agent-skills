---
name: workflow
description: "Default entry point for any new feature or requirement with no existing design. Trigger: 'add X to the API', 'build X', 'new requirement', any problem where the solution isn't designed yet. Runs design → task breakdown in sequence. Do NOT trigger if design already confirmed — use task-breakdown. Do NOT trigger if user only wants a spec — use design."
---

# Workflow: Design → Task Breakdown

This skill orchestrates the full pipeline. It does not duplicate the other skills — it sequences them and enforces the gates between phases.

## The Pipeline

```
Requirement Clarification (design skill)
  ↓ interview → goals/constraints → spec confirmed
  ↓ spec produces the design (approaches, pros/cons) → design confirmed
Task Breakdown (task-breakdown skill)
  ↓ user confirms tasks
Jira (optional, via task-breakdown)
```

Each phase gates the next. No skipping.

## How to Run

### Step 0: Derive the feature name

Before the first interview question, derive a feature slug from the user's initial description. Use lowercase, hyphen-separated words (e.g., `enable-inorch-get-pricing`, `cart-checkout-validation`). This becomes the output filename. You can refine it after the spec is confirmed if the initial description was vague.

### Step 1: Requirement Clarification & Design

Follow the **design** skill end-to-end:
1. Explore status quo
2. Interview (one question at a time, with recommendations)
3. Goals / Constraints / Non-goals
4. GIVEN/WHEN/THEN spec — **Gate: user confirms spec before design begins**
5. Approaches with pros/cons — **Gate: user confirms chosen approach**

**After spec is confirmed:** Write `docs/<feature-slug>.md` with:
```
# <Feature Name>

## Background
[status quo and motivation]

## Goals & Constraints
[goals, constraints, non-goals]

## Spec
[GIVEN/WHEN/THEN]
```

**After design is confirmed:** Append to the file:
```
## Design
[approaches considered, chosen approach, rationale, lifecycle/removal plan]
```

### Step 2: Task Breakdown

Feed the confirmed design into the **task-breakdown** skill:
1. Analyze the design output from Step 1
2. Build dependency graph
3. Define tasks (self-contained, max 10 files, TDD, incremental commits)
4. Review with user

**Gate:** User confirms the task list.

**After tasks are confirmed:** Append to the file:
```
## Task Breakdown
[dependency graph, then full task template for every task]
```

### Step 3: Jira (Optional)

Ask user if they want tasks created in Jira. If yes, follow the task-breakdown skill's Jira integration phase.

## Rules

- Run skills in order. Spec gates design. Design gates task breakdown. Reversing or skipping breaks the chain.
- Each gate requires explicit user confirmation. "Looks good" counts. Silence does not.
- Write to file after each gate — not all at once at the end. If the session is interrupted, partial progress is preserved.
- If the user wants to revisit a previous phase mid-flow (e.g., "wait, I want to change the spec"), go back. Update the spec in the file, then re-run task breakdown with the updated design. Don't patch — regenerate.
- If the user joins mid-pipeline (e.g., already has a design), skip completed phases. Start from the first incomplete phase.
- The output file is the deliverable. It should be complete enough that someone reading it cold understands the problem, the decision, and the implementation plan without needing the conversation history.
