---
id: t_UlglCJ
slug: integration-test-rolling-restart
title: Integration test — induced rolling restart leaves no stranded claims
workgraph: vafi-rolling-restart-fix
order: 3
required_tags:
  - executor
depends_on:
  - t_4ZZ60T
  - t_QrwliD
target_repo: viloforge/vafi
agent_model: claude-opus-4-7
judge: true
created_at: 2026-05-14T00:00:00Z
acceptance_criteria: []  # TBD — to be filled by spec-author
---

# Spec

TBD — to be spec-authored.

Test fixture in vafi repo that exercises an induced rolling restart
of an executor Deployment with a task in `doing`, verifying that
no task remains stranded.

Spec-author phase resolves: which test environment (vafi-dev cluster
vs ephemeral kind/k3d in CI), the "claim-and-block" harness mode
(test-only flag or a real long-sleep task), exact assertion shape
(poll vtaskforge until `doing` count for the dying pod's agent ID
drops to zero, or timeout), wait windows, cleanup. Test MUST exercise
both halves: (a) deliver SIGTERM during a held claim → T1 path
releases it; (b) simulate a hard kill (SIGKILL or terminationGracePeriodSeconds=0)
followed by new pod startup → T2 path releases it.

# References

- workgraph.md (this directory)
- plan.md (this directory) §"T3 — Integration test"
- 01-impl-sigterm-graceful-release.md (sibling)
- 02-impl-startup-reconciliation.md (sibling)
