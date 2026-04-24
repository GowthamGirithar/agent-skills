---
name: workflow
description: "Default skill for any new feature or requirement. If the user describes a problem, feature, or change they want — and no design or spec exists yet — use this skill. Trigger examples: 'we need to add caching to the API', 'users are requesting dark mode', 'I want to refactor the auth system', 'let's build X', 'new feature', 'I have a requirement', or any problem description where the solution is not yet designed. This is the entry point for all implementation work that hasn't been planned yet. Do NOT trigger if the user already has a confirmed design/spec and just wants tasks — that's task-breakdown. Do NOT trigger if the user explicitly says they only want the design/spec without tasks — that's requirement-design."
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
