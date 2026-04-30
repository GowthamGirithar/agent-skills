# Agent Skills

A collection of skills that enforce a structured software engineering workflow: from requirement clarification through design, task breakdown, and test-driven implementation — with per-feature stage documents, a manifest for status tracking, and a changelog for traceability.

## Skills

### workflow

**Entry point for new work.** Orchestrates the full pipeline: design → task breakdown → (optional) Jira. Creates a per-feature folder with one document per stage (`spec.md`, `design.md`, `tasks.md`), a `manifest.md` status tracker, and a `changelog.md`. Documents are updated in-place — git tracks the diff history.

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
      → Step 0: Derive feature slug, create folder + manifest + changelog
      → design
          1. Explore status quo
          2. Interview (one question at a time)
          3. Goals / Constraints / Assumptions / Non-goals
          4. Spec (GIVEN/WHEN/THEN) → write spec.md, manifest: Spec=confirmed
          5. Design (approaches, pros/cons) → write design.md, manifest: Design=confirmed
      → task-breakdown
          1. Analyze design
          2. Build dependency graph
          3. Define tasks (self-contained, max 10 files)
          4. Review with user → write tasks.md, manifest: Tasks=confirmed
          5. Jira integration (optional)
      → tdd (per task)
          RED → GREEN → REFACTOR → commit → repeat
```

## Feature Folder Structure

```
docs/<feature-slug>/
├── manifest.md     # status tracker (single source of truth for stage state)
├── spec.md         # requirement specification
├── design.md       # design approaches and chosen solution
├── tasks.md        # task breakdown and dependency graph
└── changelog.md    # chronological log of meaningful changes
```

Documents are updated in-place. The file always reflects the current state. Git tracks the full diff history; the changelog captures *why* changes happened.

## Status Lifecycle

Each stage in the manifest progresses through three statuses:

```
pending → draft → confirmed
```

- **pending** — Stage hasn't started yet
- **draft** — Output exists but not yet confirmed by user
- **confirmed** — User explicitly approved. Gates the next phase

A stage moves to `confirmed` only with explicit user approval. "Looks good" counts. Silence does not.

When all three stages reach `confirmed`, the feature is ready for implementation.

## Cascade Rule

Changes flow downstream. When an upstream stage changes, check whether downstream stages still hold:

- **Spec changed** → Does the design still hold? If not, update `design.md` too
- **Design changed** → Do the tasks still reflect the design? If not, update `tasks.md` too

Updated downstream stages drop back to `draft` and need explicit confirmation again. Stages that remain valid don't need changes — note in the changelog why they're unaffected.

## Structure

```
skills/
├── workflow/SKILL.md        # Pipeline orchestrator + manifest/changelog management
├── design/SKILL.md          # Requirement clarification & design
├── task-breakdown/SKILL.md  # Task splitting & Jira integration
└── tdd/SKILL.md             # Test-driven development cycle
```
