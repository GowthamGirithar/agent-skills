# Agent Skills

A collection of skills that enforce a structured software engineering workflow: from requirement clarification through design, task breakdown, and test-driven implementation — with per-feature stage documents, a manifest for status tracking, and a changelog for traceability.

## Installation

### Option 1 — Claude Code plugin (recommended)

Read-only, always-current. Install once, then pull updates with `/plugin update`.

```
/plugin marketplace add GowthamGirithar/agent-skills
/plugin install agent-skills@gowtham
```

Or from the shell:

```bash
claude plugin marketplace add GowthamGirithar/agent-skills
claude plugin install agent-skills@gowtham
```

### Option 2 — `skills` CLI (cross-agent, editable)

A generic installer that works with 20+ agents (Claude Code, Cursor, Copilot,
and more). Pick the skills and target agent interactively; it copies editable
files into your project.

```bash
npx skills@latest add GowthamGirithar/agent-skills
```

### Option 3 — Copy into a repo manually (editable)

Copies the skill folders into a project so you can customize them. Claude Code
discovers skills under `.claude/skills/`, so copy there:

```bash
git clone https://github.com/GowthamGirithar/agent-skills /tmp/agent-skills
mkdir -p .claude/skills
cp -R /tmp/agent-skills/skills/* .claude/skills/
```

Use `~/.claude/skills/` instead of `.claude/skills/` to install for your user
across all projects. Commit `.claude/skills/` to share with your team.

## Skills

### workflow

**Entry point for new work.** Orchestrates the full pipeline, sequencing the other skills and enforcing a confirmation gate between every phase — no skipping. Each feature gets a folder with one document per stage (`spec.md`, `design.md`, `tasks.md`), a `manifest.md` status tracker, and a `changelog.md`; documents are updated in-place and git tracks the diff history.

```
requirement (user)
  → Step 0: derive feature slug → create folder + manifest + changelog
  → design skill
      1. Explore status quo
      2. Interview (one question at a time, with recommendations)
      3. Goals / Constraints / Assumptions / Non-goals
      4. Spec (GIVEN/WHEN/THEN) → gate: user confirms → write spec.md, manifest: Spec=confirmed
      5. Approaches (pros/cons) → gate: user confirms → write design.md, manifest: Design=confirmed
  → task-breakdown skill
      1. Analyze the confirmed design
      2. Build dependency graph
      3. Define tasks (self-contained, max 10 files)
      4. Review → gate: user confirms → write tasks.md, manifest: Tasks=confirmed
  → Implementation choice: implement locally or create Jira tickets?
      ├── Local: branch → tasks.json → per-task RED → GREEN → REFACTOR → verify → commit
      └── Jira: create tickets (task-breakdown skill), then stop
  → Revision: on any post-confirmation change → update in-place, mark draft, re-confirm, cascade downstream
```

### design

Clarifies requirements and designs the solution. Runs a one-question-at-a-time interview with recommendations, surfaces assumptions and goals/constraints/non-goals, produces a GIVEN/WHEN/THEN spec, then evaluates design approaches with pros and cons.

### task-breakdown

Splits a confirmed design into self-contained, implementable tasks. Each task is a standalone work order — complete enough that an agent picks it up cold and delivers working code. Includes dependency graphs, per-task specs, documentation impact, and optional Jira integration.

### tdd

Enforces Red-Green-Refactor TDD for all implementation work. Guides the cycle: write a failing test (RED), implement the minimum code to pass (GREEN), clean up (REFACTOR). Covers test naming (DAMP), Arrange-Act-Assert structure, and language-specific conventions.
