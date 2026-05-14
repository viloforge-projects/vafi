---
id: t_QrwliD
slug: impl-startup-reconciliation
title: Release orphaned 'doing' claims on controller startup
workgraph: vafi-rolling-restart-fix
order: 2
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vafi
agent_model: claude-opus-4-7
judge: true
created_at: 2026-05-14T00:00:00Z
acceptance_criteria: []  # TBD — to be filled by spec-author
---

# Spec

TBD — to be spec-authored.

The architectural design is in
`viloforge-platform/docs/vafi-runtime-DESIGN.md` §"The
startup-reconciliation invariant". Spec-author phase resolves:
exact insertion point after `work_source.register(...)` in
`controller/controller.py`, SDK call shape for
`client.agents.tasks(agent_id, status="doing")`, per-task unclaim
error handling, logging conventions, unit-test shape (mock
`agents.tasks` returns N stranded claims, assert N unclaim calls
fire).

Must NOT regress in the multi-replica/shared-agent-ID case beyond
what is already documented in the workgraph "Out of scope" — i.e.,
the deployment is assumed to be max-1-active-replica-per-agent-ID,
and reconciliation is unsafe outside that shape.

# References

- workgraph.md (this directory)
- plan.md (this directory)
- viloforge-platform/docs/vafi-runtime-DESIGN.md §"The startup-reconciliation invariant"
- vafi/src/controller/controller.py:81 (register call site)
- vtaskforge SDK: `client.agents.tasks(agent_id, status='doing')`,
  `client.tasks.unclaim(task_id)`
