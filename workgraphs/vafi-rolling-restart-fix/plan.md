---
authored_by: operator:lonvilo
co_authors:
  - claude-opus-4-7
created_at: 2026-05-14T00:00:00Z
last_revision: 2026-05-14T00:00:00Z
---

# Approach

Two independent client-side changes in the vafi controller, validated
together by an induced-rolling-restart integration test.

**T1 — SIGTERM graceful release.** Extend the existing signal
handler in `controller/controller.py:_setup_signal_handlers` (which
today only sets `self._shutdown`). On signal:

1. Set `self._shutdown` (existing).
2. If a current claim is being held, call
   `client.tasks.unclaim(current_claim_id)`.
3. Let the existing shutdown path finish heartbeats and exit.

The controller must track the current claim ID. The poll loop at
`controller/controller.py:164` already assigns `claimed_task = await
self.work_source.claim(...)`; spec-author phase will determine the
cleanest place to expose this as a release-time-readable attribute.

**T2 — Startup reconciliation.** After agent registration (around
`controller/controller.py:81`), before entering the claim loop:

1. Call `client.agents.tasks(self._agent_info.id, status="doing")`
   (existing SDK manager; backed by `GET /v2/agents/<id>/tasks/`).
2. For each returned task, log + call `client.tasks.unclaim(task.id)`.
3. Errors on individual unclaim calls: log loudly, continue
   (best-effort; Heartbeat Timeout worker is the server-side
   backstop).
4. Proceed into the claim loop normally.

**T3 — Integration test.** Test fixture in vafi repo. Pseudocode:

1. Deploy `executor-pi` to a test namespace; wait for ready.
2. Submit a task that intentionally blocks (e.g., a long-running
   sleep harness invocation, or a test-mode flag that holds the
   claim for N seconds).
3. Wait for the executor to claim it (`doing`).
4. `kubectl rollout restart deployment/executor-pi -n <ns>`.
5. Wait `terminationGracePeriodSeconds + 30s`.
6. Assert: task is back in `todo` (or claimed by the new pod);
   no task is stranded in `doing` claimed by a non-existent pod.

# Considered alternatives

**Alt A — Use `tasks.reset` instead of `tasks.unclaim`.**
Rejected. `reset` is documented as an admin force-transition that
bypasses the state machine; using it from an agent for normal
operational claim release is a layering violation. `unclaim` is
the agent-grade endpoint, state-machine-respecting, with a dedicated
`unclaimed` event for audit. (The handoff/runtime-DESIGN initially
proposed `reset` / a not-yet-built `release` endpoint; investigation
during this architect session found `unclaim` already exists with
exactly the right semantics. Runtime DESIGN updated accordingly.)

**Alt B — Add a new `POST /v2/tasks/<id>/release/` endpoint with a
`reason` field.** Rejected for Phase 1. Would force a multi-repo
workgraph (violates the Phase 1 single-repo constraint stated in
`implementation-roadmap-PLAN.md` §"Phase 1"). Server-side
distinction between SIGTERM-release and startup-recon is currently
a nice-to-have only — pod logs differentiate them. Tracked as a
future refinement note in `vafi-runtime-DESIGN.md`.

**Alt C — Split each impl task into `design-X` + `impl-X` siblings**
(matching the worked example in `project-repo-DESIGN.md`). Rejected.
The architectural design for both halves already exists in
`vafi-runtime-DESIGN.md`. A separate design task would duplicate
spec-author work and add ceremony without methodology value at the
bugfix tier. This becomes our first **methodology finding** to
surface in `completion.md`: *for bugfix-tier work where
architectural design pre-exists, prefer flat DAG.*

**Alt D — Server-side claim-TTL based on heartbeat freshness.**
Rejected as the primary fix. Heartbeat-TTL is a server-side
backstop that already exists implicitly (referenced in
`vafi-runtime-DESIGN.md` Phase 5 fallback). It catches edge cases
the client-side fix misses (node death with no SIGTERM, pre-fix
versions), but is slow (waits for TTL) and centralizes failure
recovery server-side. The client-side fix is the operationally
correct primary path.

**Alt E — Ship T1 only; defer T2 to a follow-up workgraph.**
Rejected. The issue body explicitly names both halves as needed,
and the more impactful half is T2 (covers SIGTERM-failed cases,
not just SIGTERM-delivered cases). Shipping only T1 would close
the easy-case gap while leaving the hard-case gap open — not a
defensible Phase 1 deliverable.

# Open questions resolved during planning

- **Q: Which release endpoint?** A: `POST /v2/tasks/<id>/unclaim/`
  (SDK: `client.tasks.unclaim`). State-machine-respecting,
  agent-grade, emits a dedicated `unclaimed` event. Verified to
  exist at `vtaskforge/src/tasks/views.py:269`. The handoff's
  `tasks.reset(...)` and the runtime DESIGN's `/release` were both
  worse choices than what already exists.
- **Q: Which "list my active claims" endpoint?** A:
  `GET /v2/agents/<id>/tasks/?status=doing` (SDK: `client.agents.tasks`).
  Verified at `vtaskforge/src/agents/views.py:85`. Note that this
  filters by `claimed_by=agent.user` (user-level), not by agent
  ID. Safe under the deployment shape vafi#4 cares about
  (max-1-active-replica-per-agent-ID rolling restart). Documented
  in workgraph.md "Out of scope" and as a constraint in the runtime
  DESIGN.
- **Q: Split design and impl per the worked example?** A: No, flat
  3-task DAG. See Alt C above.
- **Q: `kind: bugfix` or `kind: feature`?** A: `bugfix`. The roadmap
  is prescriptive; the DESIGN worked example was illustratively
  `feature` but has been updated to `bugfix` to match.

# Risks

- **Risk 1: SIGTERM handler runs in a non-async signal context** but
  the SDK call is async. Need to schedule the unclaim coroutine on
  the controller's event loop from the signal handler safely
  (asyncio.run_coroutine_threadsafe or set a flag + await in main
  loop). Spec-author phase to resolve.
- **Risk 2: Startup-reconciliation query under N>1 active replicas
  with shared agent ID** could release claims from sibling
  replicas. Out-of-scope for vafi#4 deployment shape (rolling
  restart = max-1-active); flagged in workgraph.md and runtime DESIGN.
- **Risk 3: Integration test flake** from timing in the rolling-restart
  window. Mitigated by: (a) using a deterministic "claim-and-block"
  harness mode, (b) generous post-restart wait window in assertions.

# References

- vafi#4: https://github.com/viloforge/vafi/issues/4
- observations/obs_jsWWsQ.md
- viloforge-platform/docs/vafi-runtime-DESIGN.md §"Phase 5 — Draining"
- viloforge-platform/docs/vafi-runtime-DESIGN.md §"The startup-reconciliation invariant"
- viloforge-platform/docs/implementation-roadmap-PLAN.md §"Phase 1 — Manual SDD validation"
- viloforge-platform/docs/project-repo-DESIGN.md §"Worked example"
- vtaskforge/src/tasks/views.py:269 (unclaim endpoint, verified)
- vtaskforge/src/agents/views.py:85 (agent tasks endpoint, verified)
- vafi/src/controller/controller.py:_setup_signal_handlers (existing signal hook)
- kb gotcha `5CB2E6x4` (vafi area)
