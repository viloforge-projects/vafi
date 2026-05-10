# vafi (ViloForge Agentic Fleet Infrastructure)

Autonomous AI agent fleet — agent pods that claim tasks from the task
execution engine, execute against linked code repos, and submit work
for judge review.

This is the **project workspace** for vafi work. It holds the
durable record of decisions, gotchas, observations (incoming
signals), and workgraphs (units of work) for the vafi product
domain.

## Current focus

Bootstrapping. The new ViloForge software factory architecture is
being built; this is the first project workspace provisioned. The
first workgraph slated for the new system is `vafi-rolling-restart-fix`
(addresses [vafi#4](https://github.com/viloforge/vafi/issues/4) — the
production-readiness blocker for the agent fleet).

Phase 1 of the implementation roadmap (per
[`viloforge/viloforge-platform/docs/implementation-roadmap-PLAN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/implementation-roadmap-PLAN.md))
runs vafi#4 manually through the SDD pipeline: operator + Claude
collaborate on triage, architecture, spec authoring, verification,
and execution. The artifacts produced (workgraph.md, plan.md,
per-task specs, completion.md) populate this repo and inform the
first methodology drafts in
[`vtf-methodologies`](https://github.com/viloforge/vtf-methodologies).

## How to work in this project

- **Architect agents (or operator + Claude in interactive sessions):**
  read `methodologies/architect.md` (when present) for project-specific
  conventions on top of global; design workgraphs and plans.
- **Spec-author agents:** read the architect's plan; walk the linked
  code repos; write per-task specs with file paths, GIVEN/WHEN/THEN
  acceptance criteria, and constraints.
- **Verifier agents:** check DAG topology, file paths, acceptance
  criteria coverage, methodology pinning, [NEEDS CLARIFICATION]
  markers.
- **Executor agents:** read the spec stack (project context →
  workgraph → task spec); implement in the linked code repos against
  the acceptance criteria.
- **Judges:** review submissions per-criterion, not against general
  code-quality vibes.
- **Operators:** see `decisions/` for durable architectural records,
  `gotchas/` for known pitfalls, `workgraphs/` for what's in flight.

## Linked code repos

- [`viloforge/vafi`](https://github.com/viloforge/vafi) — source
- [`viloforge/viloforge-platform`](https://github.com/viloforge/viloforge-platform) — deploy

## Pointers

- Cycle: `2026-Q2-bootstrap`
- Methodology pin: `vtf-methodologies@v0.0` (will bump as Phase 1
  populates methodology files)
- See `project.yaml` for the structured project metadata.

## Layout reference

This repo follows the conventions in
[`viloforge/viloforge-platform/docs/project-repo-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/project-repo-DESIGN.md):

```
.
├── README.md                       # this file (project narrative + intro)
├── project.yaml                    # rigid project metadata
├── decisions/                      # durable architectural records
├── gotchas/                        # documented surprises
├── observations/                   # incoming raw signals (pre/post triage)
├── workgraphs/                     # work units (DAGs of tasks)
│   └── <slug>/
│       ├── workgraph.md
│       ├── plan.md
│       ├── tasks/
│       ├── reviews.md
│       └── completion.md
├── methodologies/                  # project-scoped methodology overrides
└── docs/                           # free-form project documentation
```
