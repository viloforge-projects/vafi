---
id: obs_jsWWsQ
created_at: 2026-05-14T00:00:00Z
source:
  kind: github_issue
  url: https://github.com/viloforge/vafi/issues/4
  external_id: "4"
captured_by: operator:lonvilo
status: linked
triage:
  triaged_at: 2026-05-14T00:00:00Z
  triaged_by: operator:lonvilo
  outcome: promoted_new
  linked_workgraphs:
    - wg_gjx8mw
  reason: |
    Production-readiness blocker. Two-part fix is well-bounded
    (SIGTERM graceful release + startup reconciliation), both halves
    specced in vafi-runtime-DESIGN.md §"Phase 5 — Draining" and §"The
    startup-reconciliation invariant". Architectural design already
    exists; this workgraph is bugfix-tier execution of that design.
    Chosen as Phase 1 manual SDD validation per
    viloforge-platform/docs/implementation-roadmap-PLAN.md §"Phase 1".
---

# Observation: Rolling-restart strands in-flight 'doing' tasks (graceful-shutdown + startup-reconciliation gap)

## Symptom

When vafi pods roll (deploy, hard-refresh, node drain), tasks claimed by the old pod can get stranded in `doing` indefinitely. The new pod re-registers under the same agent ID but never resumes or releases the orphaned claim.

## Reproduction

Canary round 4 on 2026-05-10:
1. ArgoCD hard-refresh of `vafi-dev` triggered rolling restart of `executor-pi`.
2. During the brief overlap window, the old pod (`dd40630`) won the claim race for canary task `i68Fg55HCFx1jfWrjxzVn` against the new pod (`422350d`).
3. Old pod received SIGTERM before spawning the harness; the claim was never released.
4. New pod started, registered with the same agent name `executor-pi` and got the same agent ID `qKnJx2WwaTKUH_gQm_-sC`.
5. New pod's poll loop only queries:
   - `GET /v2/tasks/claimable/?tags=executor,pi`  (status=todo, by tags)
   - `GET /v2/tasks/?status=changes_requested`
   Neither matches a task in `doing`.
6. Task stuck for 39+ min in `doing` with `claimed_by` pointing at our running agent ID — no live worker. Resolved manually by `vtf task reset i68Fg55HCFx1jfWrjxzVn --status todo`.

## Why it matters

This is not a canary-only issue. Every production deploy is a rolling restart. Any task in `doing` when its pod gets SIGTERM is at risk:
- Lost work if the harness was mid-execution
- Stuck claims requiring human intervention
- Misleading dashboards/metrics ("agent X is working" — no it isn't)

## Proposed fix

Two complementary changes:

**1. Graceful shutdown — release claims on SIGTERM.**
Install a signal handler that, before exiting, releases each in-flight `doing` task back to `todo`. Best-effort; bound by `terminationGracePeriodSeconds`. Code lives in `controller/__main__.py` / `controller/controller.py` shutdown path.

**2. Startup reconciliation — recover orphaned claims.**
On controller boot, after agent registration, query tasks held in `doing` by this agent's user account. For each result, release back to `todo` so it can be re-claimed.

Both halves use the existing `POST /v2/tasks/<id>/unclaim/` endpoint (state-machine-respecting; emits dedicated `unclaimed` event).

## Out of scope (future)

- Resumable harnesses (re-attaching to a partial claude session). Phase 5+ work.
- Server-side TTL on claims based on agent heartbeat freshness. Could be a server-side complement, but client-side reconciliation is owned by the agent and is the natural place to put it.
- Dedicated `/v2/tasks/<id>/release/` endpoint with optional `reason` field (current `unclaim` has no reason payload; differentiating SIGTERM-release from startup-recon at the server is currently a nice-to-have only).
- Multi-replica fan-out under a single agent ID (current `agents/<id>/tasks/` filters by user account; safe for max-1-active-replica-per-agent-ID deployments only).

Reference gotcha: `kb load vafi` — entry `5CB2E6x4`.

# Triage notes

Promoted to workgraph `vafi-rolling-restart-fix` (`wg_gjx8mw`). Severity: production-readiness blocker. Chosen as Phase 1 manual SDD validation per `viloforge-platform/docs/implementation-roadmap-PLAN.md` §"Phase 1 — Manual SDD validation".

— triaged by operator:lonvilo at 2026-05-14T00:00:00Z
