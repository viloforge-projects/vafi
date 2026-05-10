# vafi (ViloForge Agentic Fleet Infrastructure)

Autonomous AI agent fleet — agent pods that claim tasks from the task
execution engine, execute against linked code repos, and submit work
for judge review.

This is the **project workspace** for vafi work. It holds the
durable record of decisions, gotchas, observations (incoming
signals), and workgraphs (units of work) for the vafi product
domain.

## Current focus

**Phase 0 complete (2026-05-10). Phase 1 next: architect session for
vafi#4.**

### Where we are

Phase 0 of the implementation roadmap finished today. Specifically:

- All platform repos transferred from operator's personal account
  (`vilosource/`) to the `viloforge` GitHub org. `viloforge/vafi`,
  `viloforge/viloforge-platform`, `viloforge/vtaskforge` all live
  there now. Auto-redirect handles old URLs.
- `viloforge-projects/` org created; **this repo** scaffolded.
- `viloforge/vtf-methodologies@v0.0` created as empty stub.
- Cluster reconfigured: ArgoCD apps, Argo Events sensors, ESO
  ExternalSecrets all updated to reference `viloforge/*` URLs and
  the new GitHub App installation (id `125572832`).
- Smoke test passed: trivial commit to `viloforge/vafi:main` →
  webhook → sensor → workflow → image build → all green.

PR #1 in `viloforge/viloforge-platform` was the YAML reconfiguration;
it's merged.

### What's next

**Phase 1 — Manual SDD validation on vafi#4.**

Open an architect session (operator + Claude). Goal: produce
`workgraphs/vafi-rolling-restart-fix/workgraph.md` and `plan.md`
through collaborative design. Then spec-author per task; verify;
implement (operator + Claude as executor); judge against acceptance
criteria; retrospector writes `completion.md`. Extract first-draft
methodology files into `vtf-methodologies` from the experience.

The workgraph addresses
[vafi#4](https://github.com/viloforge/vafi/issues/4) — the
rolling-restart claim orphaning bug — production-readiness blocker
for the vafi worker fleet. Real outcome (a fix) plus
methodology validation.

### Cluster state to be aware of

- All Phase 0-relevant ArgoCD apps are Synced + Healthy
  (`vafi-dev`, `vtf-dev`, `platform-bootstrap`, etc.).
- Pre-existing peripheral apps in non-Synced states:
  - `harbor` — OutOfSync + Degraded (was Degraded before transfer;
    not Phase 0 regression)
  - `vafi-dev-secrets` — OutOfSync + Healthy
  - `vtf-dev-secrets` — Unknown + Healthy
- These don't block Phase 1 (workloads run fine); reconcile in a
  separate operator session when convenient.

### How to resume in a future session

1. Read `viloforge/viloforge-platform/docs/INDEX.md` (entry point
   to the design set; pick the "I'm planning the build-out" reading
   path).
2. Read this section ("Current focus") to know exactly where we
   stopped.
3. Begin the architect session for vafi#4 — operator + Claude
   discussing the design. The artifact of the session is
   `workgraphs/vafi-rolling-restart-fix/workgraph.md` committed to
   this repo.

### Reference

Full Phase 1 procedure: `viloforge/viloforge-platform/docs/implementation-roadmap-PLAN.md` §"Phase 1 — Manual SDD validation".

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
