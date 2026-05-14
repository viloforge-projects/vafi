---
id: t_4ZZ60T
slug: impl-sigterm-graceful-release
title: Release in-flight claim on SIGTERM before exit
workgraph: vafi-rolling-restart-fix
order: 1
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
`viloforge-platform/docs/vafi-runtime-DESIGN.md` §"Phase 5 — Draining
(SIGTERM received)". Spec-author phase resolves: exact code location
within `controller/controller.py`, how to safely invoke an async SDK
call from a synchronous signal handler, how the current-claim ID is
tracked across the poll loop, time-budget enforcement, unit-test
shape, file paths for any new modules.

# References

- workgraph.md (this directory)
- plan.md (this directory)
- viloforge-platform/docs/vafi-runtime-DESIGN.md §"Phase 5 — Draining"
- vafi/src/controller/controller.py:_setup_signal_handlers
- vtaskforge SDK: `client.tasks.unclaim(task_id)`
