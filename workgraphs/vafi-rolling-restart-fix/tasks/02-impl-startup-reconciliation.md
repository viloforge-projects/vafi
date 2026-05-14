---
id: t_QrwliD
slug: impl-startup-reconciliation
title: Release orphaned 'doing' claims on controller startup
workgraph: vafi-rolling-restart-fix
order: 2
required_tags:
  - executor
depends_on:
  - t_cHx0UU            # T0 — needs AsyncAgentManager.tasks + AsyncTaskManager.unclaim
target_repo: viloforge/vafi
agent_model: claude-opus-4-7
judge: true
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T2-1 — After agent registration and after the agent-heartbeat
    task is started (i.e., between
    `controller.py:88-95` and the poll loop at `:97`), the controller
    SHALL invoke a new method `await self._reconcile_orphaned_claims()`
    before entering the `while not self._shutdown.is_set()` loop.
    Placement rationale: heartbeat must be running so the server
    doesn't time-out the agent mid-reconciliation; reconciliation
    must complete before the poll loop starts claiming new work
    (otherwise new claims could race with reconciliation cleanup).
  - AC-T2-2 — `_reconcile_orphaned_claims` SHALL:
    (a) call `tasks = await self.work_source.list_active_claims(self._agent_info.id)`;
    (b) for each `task` in `tasks`, call
        `await self.work_source.unclaim(task.id)`;
    (c) log each successful unclaim at INFO level with both the
        task ID and a stable phrase like `"startup-reconciliation"`
        so pod-log readers can distinguish reconciliation events
        from T1 SIGTERM-release events;
    (d) return normally regardless of how many tasks were processed
        (including zero).
  - AC-T2-3 — Individual per-task `unclaim` failures SHALL be caught,
    logged at ERROR with the task ID and exception, and reconciliation
    SHALL continue with the remaining tasks. A single bad task MUST
    NOT block reconciliation of the others.
  - AC-T2-4 — A failure in the initial `list_active_claims` call
    (network, vtaskforge unavailable, authorization, etc.) SHALL be
    caught, logged at ERROR, and the controller SHALL continue to
    the poll loop. Reconciliation is best-effort; the server-side
    Heartbeat Timeout worker is the documented backstop
    (`vafi-runtime-DESIGN.md` Phase 5 fallback). The controller MUST
    NOT crash on reconciliation failure.
  - AC-T2-5 — The `WorkSource` protocol gains a new method:
    `async def list_active_claims(self, agent_id: str) -> list[TaskInfo]`.
    Placed adjacent to `claim` / `unclaim` (T1's addition) in
    protocol order. The `VtfWorkSource` implementation calls
    `result = await self._client.agents.tasks(agent_id, status="doing")`
    and returns `[self._sdk_task_to_info(t) for t in result.items]`
    (using the existing `_sdk_task_to_info` helper at
    `vtf.py:197`). The `client.agents.tasks` method is provided by
    T0 (`t_cHx0UU`).
  - AC-T2-6 — A pytest-asyncio unit test verifies the happy path:
    `MockWorkSource.list_active_claims` returns three `TaskInfo`
    objects → exactly three `unclaim` calls fire with the correct
    IDs in the order returned. Test asserts the controller proceeds
    into the poll loop after reconciliation (no early exit).
  - AC-T2-7 — A second pytest-asyncio unit test verifies error
    isolation: `list_active_claims` returns three tasks; the mock's
    `unclaim` is configured to raise for the *second* task. The
    test asserts: (a) `unclaim` was called for all three IDs;
    (b) the controller still proceeded to the poll loop;
    (c) the error was logged at ERROR (caplog fixture inspection
    is acceptable).
  - AC-T2-8 — A third pytest-asyncio unit test verifies the
    list-call-failure isolation: `list_active_claims` raises;
    `unclaim` is NEVER called; controller proceeds to poll loop;
    error logged at ERROR.
  - AC-T2-9 — Multi-replica safety constraint MUST be documented in
    a docstring on `_reconcile_orphaned_claims`: "Safe only under
    max-1-active-replica-per-agent-ID deployments. Server-side
    filter is by `claimed_by=agent.user`, not by agent-instance,
    so reconciling under N>1 active replicas with shared agent ID
    would release claims still being worked on by sibling
    replicas." (See workgraph.md "Out of scope" + runtime DESIGN
    safety constraint.)
---

# Spec

## Files touched

- `src/controller/controller.py` — new `_reconcile_orphaned_claims`
  method on `Controller`; one new call site in `run()` between the
  heartbeat-task start and the poll loop.
- `src/controller/worksources/protocol.py` — add
  `list_active_claims` to the `WorkSource` Protocol.
- `src/controller/worksources/vtf.py` — implement `list_active_claims`
  using `self._client.agents.tasks(agent_id, status="doing")` (T0
  provides this SDK method).
- `tests/test_controller.py` — extend `MockWorkSource` with a
  `list_active_claims` method + per-task-result configuration; add
  three new tests in `TestController`.

NOT touched: T1's files (`__init__`, `_poll_and_execute`,
`finally` block in `run()`). T2 only adds *new* code on top of T1;
it doesn't modify T1's surface. (The `unclaim` call in T2 is the
same `WorkSource.unclaim` T1 introduced; T2 reuses that method.)
This separation lets T1 and T2 be reviewed independently and
keeps the methodology-extraction signal clean.

## controller.py changes

### New method `_reconcile_orphaned_claims`

Add near the other private helpers (e.g., adjacent to
`_setup_signal_handlers` at line 532, or just before it for
locality with `run()`'s call sites — match the file's existing
organization):

```python
async def _reconcile_orphaned_claims(self) -> None:
    """Release any tasks still claimed by this agent from a prior pod.

    Called once during controller startup, after agent registration
    and after the agent-heartbeat task is running, but before the
    poll loop starts. Queries vtf for tasks this agent's user
    account still holds in `doing` and unclaims each, recovering
    from ungraceful prior-pod exits (OOM, node death, pre-fix
    versions that lacked the T1 SIGTERM handler).

    Safety: this method assumes at most one active replica per
    agent ID at any time. The server-side endpoint filters by
    `claimed_by=agent.user`, not by agent-instance, so reconciling
    under N>1 active replicas sharing an agent ID would release
    claims still being worked on by sibling replicas. Deployments
    that need multi-replica fan-out under a shared agent ID
    require server-side support not present today.

    All errors are caught and logged; reconciliation is
    best-effort. The server-side Heartbeat Timeout worker is the
    documented backstop for claims this method fails to release.
    See vafi-runtime-DESIGN.md §"The startup-reconciliation
    invariant" and §"Phase 5 — Draining" for the broader design.

    Implements vafi#4 T2 (startup-reconciliation half of the fix).
    """
    assert self._agent_info is not None, \
        "_reconcile_orphaned_claims must be called after register()"

    try:
        active_claims = await self.work_source.list_active_claims(
            self._agent_info.id
        )
    except Exception as e:
        logger.error(
            f"startup-reconciliation: failed to list active claims "
            f"for agent {self._agent_info.id}: {e}",
            exc_info=True,
        )
        return

    if not active_claims:
        logger.info("startup-reconciliation: no orphaned claims found")
        return

    logger.warning(
        f"startup-reconciliation: found {len(active_claims)} orphaned "
        f"claim(s) from prior pod incarnation; releasing"
    )

    for task in active_claims:
        try:
            await self.work_source.unclaim(task.id)
            logger.info(
                f"startup-reconciliation: released orphaned claim "
                f"on task {task.id}"
            )
        except Exception as e:
            logger.error(
                f"startup-reconciliation: failed to unclaim task "
                f"{task.id}: {e}",
                exc_info=True,
            )
            # Continue with remaining tasks — per-task failure must
            # not block reconciliation of siblings.
```

### Call site in `run`

Existing code at lines 86-98 ends with:

```python
logger.info(f"Registered as agent {self._agent_info.id}")

# Start agent-level heartbeat loop (runs always, independent of tasks)
agent_hb_task = asyncio.create_task(
    agent_heartbeat_loop(
        self.work_source,
        self._agent_info.id,
        self.config.poll_interval,
    )
)
logger.info("Started agent heartbeat loop")

# Main polling loop
logger.info("Starting poll loop...")
while not self._shutdown.is_set():
```

Insert the reconciliation call between `logger.info("Started agent heartbeat loop")`
and `logger.info("Starting poll loop...")`:

```python
logger.info("Started agent heartbeat loop")

# vafi#4 T2: release any claims this agent's user holds in `doing`
# from a prior pod incarnation that didn't release gracefully.
# Runs ONCE at startup. Best-effort — see method docstring.
await self._reconcile_orphaned_claims()

# Main polling loop
logger.info("Starting poll loop...")
while not self._shutdown.is_set():
```

The `await` is safe because we're already in `run()`'s async
context. Reconciliation runs inline; any time it takes is bounded
by the SDK transport timeout × number of orphaned claims (which
should be 0 or 1 in healthy deployments).

## protocol.py changes

Add the method to `WorkSource`, adjacent to `claim` / `unclaim`:

```python
async def list_active_claims(self, agent_id: str) -> list[TaskInfo]:
    """List tasks currently claimed by this agent's user account.

    Used by the startup-reconciliation path to discover orphaned
    claims left by a previous pod incarnation. The implementation
    filters by status='doing'.

    Args:
        agent_id: Agent ID. The implementation may translate this
            to the agent's user account for the query.

    Returns:
        List of TaskInfo for all `doing` tasks claimed by this
        agent's user. Empty list if none.
    """
    ...
```

## vtf.py changes

Implement, placed adjacent to the new `unclaim` method T1 added
(around line 76):

```python
async def list_active_claims(self, agent_id: str) -> list[TaskInfo]:
    """List `doing` tasks held by this agent's user. See protocol docstring."""
    result = await self._client.agents.tasks(agent_id, status="doing")
    return [self._sdk_task_to_info(t) for t in result.items]
```

`result.items` matches the `PagedResult` shape used elsewhere in
this file (e.g., the existing `poll` method at line 44+ uses
`claimable_result.items`). Reuse the existing
`_sdk_task_to_info` helper at `vtf.py:197` to convert SDK
`Task` objects to controller `TaskInfo`.

If the result is paginated and there are more than the default
page size (50) of orphaned claims, only the first page is
processed this iteration. This is acceptable for vafi#4 because:
(a) realistically a healthy deployment has 0-1 orphaned claims at
any time, not 50+; (b) the next pod restart will reconcile the
remainder; (c) the server-side Heartbeat Timeout backstop catches
any that escape forever. Pagination handling is explicitly NOT
implemented in T2 — adding it is out of scope.

## test_controller.py changes

### Extend `MockWorkSource`

Add to the mock's state and method surface (match the existing
attribute-init pattern):

```python
# In MockWorkSource.__init__ or class attrs:
self.list_active_claims_response: list[TaskInfo] = []
self.list_active_claims_error: Exception | None = None
self.unclaim_errors: dict[str, Exception] = {}  # task_id -> exception to raise

# New method:
async def list_active_claims(self, agent_id: str) -> list[TaskInfo]:
    self.list_active_claims_calls.append(agent_id)
    if self.list_active_claims_error is not None:
        raise self.list_active_claims_error
    return self.list_active_claims_response
```

And extend the existing T1 mock `unclaim` to honor `unclaim_errors`:

```python
async def unclaim(self, task_id: str) -> None:
    self.unclaim_calls.append(task_id)
    if task_id in self.unclaim_errors:
        raise self.unclaim_errors[task_id]
```

Initialize the new tracking lists (`list_active_claims_calls`)
in the mock's `__init__` alongside the existing call-tracking
lists.

### New tests in `TestController`

**Test 1 — happy path (AC-T2-6):**

```python
async def test_reconciles_orphaned_claims_on_startup(
    self, mock_work_source, test_config, sample_agent
):
    """Startup reconciliation releases all orphaned claims (vafi#4 T2)."""
    mock_work_source.register_response = sample_agent
    mock_work_source.list_active_claims_response = [
        TaskInfo(id="task-alpha", title="Alpha", ...),
        TaskInfo(id="task-beta", title="Beta", ...),
        TaskInfo(id="task-gamma", title="Gamma", ...),
    ]
    # No poll work — controller proceeds idle after reconciliation.
    mock_work_source.poll_response = None

    controller = Controller(mock_work_source, test_config)
    run_task = asyncio.create_task(controller.run())
    await _wait_until(
        lambda: len(mock_work_source.unclaim_calls) == 3, timeout=2.0
    )
    controller._shutdown.set()
    await asyncio.wait_for(run_task, timeout=5.0)

    assert mock_work_source.unclaim_calls == [
        "task-alpha", "task-beta", "task-gamma"
    ]
    # list_active_claims called exactly once (startup only):
    assert len(mock_work_source.list_active_claims_calls) == 1
```

**Test 2 — per-task unclaim error isolation (AC-T2-7):**

```python
async def test_reconciliation_continues_on_per_task_unclaim_error(
    self, mock_work_source, test_config, sample_agent, caplog
):
    """A failed unclaim on one task doesn't block sibling unclaims."""
    mock_work_source.register_response = sample_agent
    mock_work_source.list_active_claims_response = [
        TaskInfo(id="task-alpha", ...),
        TaskInfo(id="task-beta", ...),
        TaskInfo(id="task-gamma", ...),
    ]
    mock_work_source.unclaim_errors = {"task-beta": RuntimeError("boom")}
    mock_work_source.poll_response = None

    controller = Controller(mock_work_source, test_config)
    run_task = asyncio.create_task(controller.run())
    await _wait_until(
        lambda: len(mock_work_source.unclaim_calls) == 3, timeout=2.0
    )
    controller._shutdown.set()
    await asyncio.wait_for(run_task, timeout=5.0)

    # All three were attempted, in order:
    assert mock_work_source.unclaim_calls == [
        "task-alpha", "task-beta", "task-gamma"
    ]
    # Error was logged at ERROR:
    assert any(
        rec.levelname == "ERROR" and "task-beta" in rec.message
        for rec in caplog.records
    )
```

**Test 3 — list-call failure isolation (AC-T2-8):**

```python
async def test_reconciliation_survives_list_failure(
    self, mock_work_source, test_config, sample_agent, caplog
):
    """If list_active_claims raises, controller still proceeds to poll loop."""
    mock_work_source.register_response = sample_agent
    mock_work_source.list_active_claims_error = RuntimeError("server unreachable")
    mock_work_source.poll_response = None

    controller = Controller(mock_work_source, test_config)
    run_task = asyncio.create_task(controller.run())
    # Wait a bit for the poll loop to run at least once
    await asyncio.sleep(0.1)
    controller._shutdown.set()
    await asyncio.wait_for(run_task, timeout=5.0)

    # No unclaim calls (list failed before iteration):
    assert mock_work_source.unclaim_calls == []
    # Error was logged at ERROR:
    assert any(
        rec.levelname == "ERROR" and "list active claims" in rec.message.lower()
        for rec in caplog.records
    )
    # Controller proceeded to poll (poll mock was queried):
    assert len(mock_work_source.poll_calls) >= 1
```

The `_wait_until` helper, `sample_agent` fixture, and existing
mock-tracking attributes (`poll_calls`, `register_response`) are
already conventions in `test_controller.py` — the spec defers to
the executor to match the exact local conventions for fixture
wiring.

# Constraints

- MUST NOT modify the `_setup_signal_handlers` method (T1's
  surface). T2 is purely additive over T1.
- MUST NOT modify `_poll_and_execute`. T2 adds no new behavior
  to the steady-state poll loop; it only runs once at startup.
- MUST NOT modify the Task data model or any vtaskforge
  server-side code.
- MUST NOT add pagination handling for `list_active_claims`. If
  >50 orphaned claims exist, only the first page is reconciled
  per startup; the next restart handles the rest. (Realistic max
  is 0-1 in healthy deployments.)
- MUST NOT call `_reconcile_orphaned_claims` from anywhere
  except `run()`'s startup path. Reconciliation is a once-per-
  controller-lifetime event.
- The reconciliation MUST NOT block startup indefinitely. If a
  single `unclaim` call hangs, the SDK transport timeout applies;
  the controller proceeds to the next task. No additional
  per-call timeout is added (matching T1's AC-T1-7 reasoning).

# Out of scope

- Multi-replica fan-out safety (documented in AC-T2-9 and
  workgraph.md "Out of scope").
- Pagination of `list_active_claims` results (documented above).
- Periodic reconciliation during steady-state (only runs once
  at startup; periodic re-checks are a different design with
  different cost/safety trade-offs).
- Resuming partial harness sessions for the orphaned claims (the
  longer-term work referenced in the issue body's "Out of scope
  (future)"; T2 only releases).
- Differentiating reconciliation-released-claims from
  graceful-released-claims at the vtaskforge audit layer (handled
  by log-level pod-side language only; server side has no
  reason field on unclaim).

# References

- workgraph.md (this directory) AC-3 — DAG-level "must query and
  release" criterion T2 operationalizes.
- plan.md (this directory) §"T2 — Startup reconciliation" — prose
  approach.
- 00-add-sdk-methods.md (T0) — produces
  `AsyncAgentManager.tasks` + `AsyncTaskManager.unclaim` that T2
  depends on.
- 01-impl-sigterm-graceful-release.md (T1) — introduces the
  `WorkSource.unclaim` method T2 reuses. T2 only adds
  `list_active_claims`; T1 owns `unclaim`.
- `viloforge-platform/docs/vafi-runtime-DESIGN.md` §"The startup-
  reconciliation invariant" — architectural design for this
  behavior.
- `vafi/src/controller/controller.py:81-95` — current
  register-then-heartbeat-then-poll sequence; reconciliation slots
  between heartbeat-start and poll-loop.
- `vafi/src/controller/controller.py:197` — `_sdk_task_to_info`
  helper used to convert SDK Task to TaskInfo (referenced in
  vtf.py's `list_active_claims` impl).
- `vafi/src/controller/worksources/vtf.py:44` — existing `poll`
  method (template for `result.items` shape in
  `list_active_claims`).
- `vafi/tests/test_controller.py:15` — `MockWorkSource` to extend.
- `vafi/tests/test_controller.py:298` —
  `test_agent_heartbeat_runs_during_idle` (model for tests that
  exercise the startup-path-then-shutdown shape).
- vtaskforge `agents/views.py:85` — server-side endpoint (filters
  by `claimed_by=agent.user`).
