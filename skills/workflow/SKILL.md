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

## Document Management

Every feature gets a dedicated folder with one file per stage. Documents are updated in-place — git tracks the full diff history. A lightweight changelog captures why changes happened so you can understand a feature's evolution at a glance.

### Folder Structure

```
docs/<feature-slug>/
├── manifest.md           # status tracker
├── spec.md               # requirement specification
├── design.md             # design approaches and chosen solution
├── tasks.md              # task breakdown and dependency graph
└── changelog.md          # chronological change log
```

### Manifest

The manifest tracks the current status of each stage. It is the single source of truth for where a feature stands.

```markdown
# <Feature Name> — Manifest

| Stage  | Status    |
|--------|-----------|
| Spec   | confirmed |
| Design | draft     |
| Tasks  | pending   |
```

**Three statuses:**

- **pending** — Stage hasn't started yet
- **draft** — Output exists but not yet confirmed by user
- **confirmed** — User approved. Gates the next phase

A stage moves to `confirmed` only with explicit user approval. "Looks good" counts. Silence does not.

When all three stages reach `confirmed`, the feature is ready for implementation.

Update the manifest immediately after every status change.

### Changelog

`changelog.md` is a chronological log of every meaningful change. Append to it — never overwrite.

```markdown
# <Feature Name> — Changelog

## [Date] — [Stage] [created | updated]
Trigger: [why — user feedback, implementation discovery, new requirement]
Summary: [one-line description of what changed]
```

The changelog exists so someone scanning the feature folder can quickly understand how it evolved without reading git diffs.

### Updating Documents

When something changes (user feedback, implementation discovery, new requirement):

1. Update the affected file in-place — the file always reflects the current state
2. Set the stage's status back to `draft` in the manifest
3. Append a changelog entry explaining what changed and why
4. **Check downstream impact** (the cascade rule — see below)
5. Get user confirmation → status back to `confirmed`

### The Cascade Rule

Changes flow downstream. When an upstream stage changes, check whether downstream stages still hold:

- **Spec changed** → Does the design still hold? If not, update `design.md` too
- **Design changed** → Do the tasks still reflect the design? If not, update `tasks.md` too

For each downstream stage that needs updating:
- Update it in-place
- Set its status to `draft`
- Add a changelog entry noting it was a cascade
- Get user confirmation before marking `confirmed` again

Stages that remain valid don't need changes — note in the changelog why they're unaffected.

## How to Run

### Step 0: Derive the Feature Name and Create Folder

Before the first interview question:

1. Derive a feature slug from the user's initial description. Use lowercase, hyphen-separated words (e.g., `enable-inorch-get-pricing`, `cart-checkout-validation`).
2. Create the feature folder:
   ```
   docs/<feature-slug>/
   ├── manifest.md      (all stages: pending)
   └── changelog.md     (empty, initialized with feature name header)
   ```
3. You can refine the feature name after the spec is confirmed if the initial description was vague — rename the folder and update paths.

### Step 1: Requirement Clarification & Design

Follow the **design** skill end-to-end:
1. Explore status quo
2. Interview (one question at a time, with recommendations)
3. Goals / Constraints / Non-goals
4. GIVEN/WHEN/THEN spec — **Gate: user confirms spec before design begins**
5. Approaches with pros/cons — **Gate: user confirms chosen approach**

**After spec is confirmed:**

Write `spec.md`:
```markdown
# <Feature Name> — Spec

## Background
[status quo and motivation]

## Goals & Constraints
[goals, constraints, non-goals]

## Assumptions
[assumptions surfaced during interview, confirmed by user]

## Spec
GIVEN [precondition]
WHEN [action]
THEN [outcome]
[all scenarios: happy path, edge cases, errors, boundaries]
```

Update manifest: Spec → `confirmed`.
Append to changelog.

**After design is confirmed:**

Write `design.md`:
```markdown
# <Feature Name> — Design

## Approaches Considered
[all approaches with pros/cons]

## Chosen Approach
[which approach and rationale]

## Risks & Open Questions
[top risks from the critic pass]

## Lifecycle / Removal Plan
[if applicable]
```

Update manifest: Design → `confirmed`.
Append to changelog.

### Step 2: Task Breakdown

Feed the confirmed design into the **task-breakdown** skill:
1. Analyze the design output from Step 1
2. Build dependency graph
3. Define tasks (self-contained, max 10 files, TDD, incremental commits)
4. Review with user

**Gate:** User confirms the task list.

**After tasks are confirmed:**

Write `tasks.md`:
```markdown
# <Feature Name> — Tasks

## Dependency Graph
[tracks and dependencies]

## Tasks
[full task template for every task]
```

Update manifest: Tasks → `confirmed`.
Append to changelog.

### Step 3: Jira (Optional)

Ask user if they want tasks created in Jira. If yes, follow the task-breakdown skill's Jira integration phase.

### Step 4: Revision

This step handles changes after any stage is confirmed. Triggers include:
- User feedback ("actually, we need to handle X differently")
- Implementation discoveries ("this approach won't work because...")
- Review findings ("the spec missed this edge case")

**Revision flow:**

1. Identify which stage is affected (spec, design, or tasks)
2. Read the current file for that stage
3. Make the changes following the relevant skill (design skill for spec/design changes, task-breakdown skill for task changes)
4. Update the file in-place
5. Update manifest: set status to `draft`
6. Get user confirmation → status back to `confirmed`
7. **Apply the cascade rule:** check downstream stages for impact
   - If unaffected, note why in the changelog
   - If affected, update them, set to `draft`, get confirmation
8. Append all changes to changelog

## Resuming Mid-Pipeline

When a user comes back to an existing feature:

1. Read `manifest.md` to understand current state
2. Find the first stage that isn't `confirmed` — that's where to resume
3. Read the completed stage files to restore context
4. Continue the pipeline from the incomplete stage

If all stages are `confirmed` and the user wants changes, follow the Revision flow (Step 4).

## Rules

- Run skills in order. Spec gates design. Design gates task breakdown.
- Each gate requires explicit user confirmation.
- Write stage documents and update the manifest after each gate — not at the end. If the session is interrupted, partial progress is preserved.
- Documents are always updated in-place. The file always reflects the current state. Git handles the diff history.
- If the user wants to change something, update the file, mark as `draft`, confirm, then cascade downstream.
- If the user joins mid-pipeline (e.g., already has a design), skip completed phases. Read the manifest, pick up from the first incomplete stage.
- The feature folder is the deliverable. Someone reading manifest + stage files + changelog should understand the problem, the decision, the implementation plan, and how it evolved.