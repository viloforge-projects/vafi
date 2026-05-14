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

**Test environment is locked at architect level: ephemeral kind or
k3d cluster spun up in CI on every PR.** Not the vafi-dev real
cluster — see plan.md §"T3 — Integration test" for rationale.

Spec-author phase resolves: kind vs k3d choice (mechanical, pick
whichever the repo already standardizes on), CI workflow file
location, image-load mechanism, the "claim-and-block" harness mode
(test-only env flag like `VF_TEST_HOLD_CLAIM_SECONDS` or a real
long-sleep task), exact assertion shape (poll vtaskforge until
`doing` count for the dying pod's agent ID drops to zero, or
timeout), wait windows, cleanup. Test MUST exercise both halves:
(a) deliver SIGTERM during a held claim → T1 path releases it;
(b) simulate a hard kill (`kubectl delete pod --grace-period=0`)
followed by new pod startup → T2 path releases it.

# References

- workgraph.md (this directory)
- plan.md (this directory) §"T3 — Integration test"
- 01-impl-sigterm-graceful-release.md (sibling)
- 02-impl-startup-reconciliation.md (sibling)
