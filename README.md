# Agent Skills

A collection of skills that enforce a structured software engineering workflow: from requirement clarification through design, task breakdown, and test-driven implementation.

## Skills

### workflow

**Entry point for new work.** Orchestrates the full pipeline: design → task breakdown → (optional) Jira. Use when starting any new feature or requirement where the solution isn't designed yet.

Trigger: "add X to the API", "build X", "new requirement"

### design

Requirement clarification and architectural design. Runs an interview (one question at a time, with recommendations), surfaces assumptions, produces GIVEN/WHEN/THEN specs, and evaluates design approaches with pros/cons.

Trigger: "clarify requirements", "write a spec", "help me design"

### task-breakdown

Splits a confirmed design into self-contained, implementable tasks. Each task is a standalone work order — complete enough that an agent picks it up cold and delivers working code. Includes dependency graphs, per-task specs, documentation impact, and Jira integration.

Trigger: "break into tasks", "create tickets", "split into work items"

### tdd

Enforces Red-Green-Refactor TDD for all implementation work. Guides the cycle: write a failing test (RED), implement minimum code to pass (GREEN), clean up (REFACTOR). Covers test naming (DAMP), Arrange-Act-Assert structure, and language-specific conventions.

Trigger: any implementation work — feature, bug fix, endpoint, function

## Pipeline

```
requirement (user)
  → workflow
      → design
          1. Explore status quo
          2. Interview (one question at a time)
          3. Goals / Constraints / Assumptions / Non-goals
          4. Spec (GIVEN/WHEN/THEN)
          5. Design (approaches, pros/cons, recommendation)
      → task-breakdown
          1. Analyze design
          2. Build dependency graph
          3. Define tasks (self-contained, max 10 files)
          4. Review with user
          5. Jira integration (optional)
      → tdd (per task)
          RED → GREEN → REFACTOR → commit → repeat
```

## Structure

```
skills/
├── workflow/SKILL.md        # Pipeline orchestrator
├── design/SKILL.md          # Requirement clarification & design
├── task-breakdown/SKILL.md  # Task splitting & Jira integration
└── tdd/SKILL.md             # Test-driven development cycle
```