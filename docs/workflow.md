# The Evolution of the Workflow

Why this workflow exists, and how it got its current shape.

## Why vibe coding and open-source spec-driven tools fell short

**Vibe coding.** Informal prompting encourages one-sentence requirements with no vital details, which reliably causes implementation failure. A structured, prompt-driven model with formal specifications closes that gap — by requiring the core intent up front, the agent can navigate the local development lifecycle systematically instead of guessing.

**Existing open-source spec-driven tools.** Trials surfaced concrete shortcomings:

- **Hallucinations and lost details** — even with precise API update details supplied, generated specs and tasks omitted them and introduced hallucinated content.
- **Decoupled testing** — unit tests were a separate, isolated task rather than integrated into a true TDD workflow.
- **No JIRA integration** — no native way to push tasks to JIRA for autonomous coding agents to pick up.
- **No local loop engineering** — no feedback loop for executing implementation tasks locally.
- **Rigid phase overlap** — tools forced multi-phase structures, when the need was to plan a single phase already decided during alignment.
- **Missing incremental commits** — no support for structured, incremental commits.

These gaps motivated building a custom agent capability instead of adopting an existing tool.

## Architectural and workflow design

### 1. Intent and codebase exploration

Early versions had the agent ask about intent before looking at the repo. This was inverted: the agent must explore the codebase first, before anything else.

### 2. Interview-style clarification and alignment

To prevent misaligned implementations, the workflow surfaces hidden assumptions immediately:

- **Codebase interviews** — an interview-style Q&A clarifies ambiguities found during exploration.
- **Goal and constraint mapping** — goals and constraints are listed explicitly from the initial intent plus clarification results.
- **Given/When/Then specs** — specifications are written in strict Given/When/Then form and require explicit user confirmation, since a wrong spec guarantees a flawed design.

### 3. Comprehensive design and guardrails

- **Explicit architectural options** — the agent is forced to present multiple approaches with a recommendation, so it can't default to the laziest path when a cleaner re-architecture exists.
- **The dev advocate (critic prompt)** — a fresh-context critic prompt reviews the design before it reaches the user, keeping the architecture simple and cost-effective without switching models, and keeping the human loop efficient.
- **Risk and documentation harvesting** — technical risks are listed explicitly; undocumented past decisions on brownfield projects are captured as ADR candidates for review; harness struggles with specific patterns are documented as an `agents.md`/`CLAUDE.md` note.
- **Explicit gates and cost optimization** — output uses a dense, filler-free style to control token cost. Confirmation gates are explicit: the model may never assume a design is confirmed just because review is delayed. User confirmation is mandatory before task breakdown.

## Task breakdown

- Task counts are derived mathematically from file sizes to gauge structural viability.
- **TDD by default** — every task enforces Red → Green → Refactor.
- **Uniform task template** — every task defines Goal, Spec, Hierarchy/Dependencies, and Verification.
- **Granular isolation** — the master spec is broken down per task so long-running feature work doesn't lose context.

## JIRA integration and local implementation

- **Automated ticket management** — the agent syncs to JIRA directly and tracks tickets in workspace docs; a changed plan updates the existing ticket instead of delete-and-recreate.
- **Autonomous local implementation** — tasks are hyper-focused micro-units, so local implementation runs unsupervised.
- **The RALPH loop** — local execution follows: `RED → GREEN → REFACTOR → VERIFY → COMMIT`.
- **Orchestrator + per-task sub-agent** — the main session orchestrates and delegates each task to a fresh sub-agent (one task at a time, sequential), so every task runs in a lean, isolated context that's discarded on return — preventing bloat/degradation over long sessions. Progress and cross-task learnings are tracked in `tasks.json`.

## Artifacts and documents generated

| Document | Purpose |
|---|---|
| `manifest.md` | High-level status of the entire workflow |
| `design.md` | Finalized design and specs — source of truth for later revisions |
| `tasks.md` | Local task breakdown — easy to alter if the design shifts |
| `changelog.md` | Chronological record of every change |

## Future roadmap
- **Automated code review/ Verifier ** — deep, agentic peer review built into the cycle.
