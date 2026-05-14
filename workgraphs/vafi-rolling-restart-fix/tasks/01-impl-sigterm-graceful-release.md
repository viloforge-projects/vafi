---
id: t_4ZZ60T
slug: impl-sigterm-graceful-release
title: Release in-flight claim on SIGTERM before exit
workgraph: vafi-rolling-restart-fix
order: 1
required_tags:
  - executor
depends_on:
  - t_cHx0UU            # T0 — AsyncTaskManager.unclaim must exist before T1 can call it
target_repo: viloforge/vafi
agent_model: claude-opus-4-7
judge: true
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T1-1 — Controller SHALL track the ID of any successfully-claimed
    task in `self._current_claim_id: str | None`, initialized to `None`
    in `__init__`. The field is set immediately after the SDK claim
    succeeds and cleared after the task reaches a terminal status
    (`complete`/`fail`/`error-during-processing`).
  - AC-T1-2 — On shutdown (whether triggered by SIGTERM, SIGINT, or
    normal loop exit), if `self._current_claim_id` is not `None`, the
    controller MUST call `self.work_source.unclaim(self._current_claim_id)`
    BEFORE calling `self.work_source.set_agent_offline(...)`.
  - AC-T1-3 — Unclaim call failures during shutdown MUST be logged at
    ERROR level with the task ID and the exception. The controller
    MUST still proceed through the rest of the shutdown sequence
    (set_agent_offline, log "Shutting down controller", return) —
    unclaim is best-effort; the server-side Heartbeat Timeout worker
    is the documented backstop.
  - AC-T1-4 — The `WorkSource` protocol gains
    `async def unclaim(self, task_id: str) -> None`, placed alongside
    `claim` in the protocol surface. The `VtfWorkSource` implementation
    calls `await self._client.tasks.unclaim(task_id)`. This relies on
    T0 (`t_cHx0UU`) having merged the `AsyncTaskManager.unclaim` method
    into `vtf-sdk-python`; T1's PR includes the `pyproject.toml`
    pin-bump to the T0 merge commit (per T0 AC-T0-6). No other
    `WorkSource` implementations exist in-tree today; if any are added
    in future they must implement this method.
  - AC-T1-5 — A `pytest-asyncio` unit test in
    `tests/test_controller.py` SHALL verify that setting
    `controller._shutdown` after a successful claim (but before the
    task reaches a terminal status) results in exactly one
    `unclaim(task_id)` call against the mock work source, with the
    correct task ID, and that the call happens before
    `set_agent_offline`. The test follows the existing
    `test_shutdown_signal_handling` / `test_marks_agent_offline_on_shutdown`
    pattern in `TestController`. The test does NOT send a real signal —
    it sets `controller._shutdown` directly, since the signal handler
    itself is one line of stdlib (`self._shutdown.set()`) with no logic
    to exercise.
  - AC-T1-6 — Mid-`execute` SIGTERM behavior is OUT OF SCOPE for T1.
    If the signal arrives while `await self.execute(claimed_task)` is
    in flight (e.g., a real harness invocation), the controller will
    exceed `terminationGracePeriodSeconds` regardless of in-pod
    handling. Recovery for that case is delegated to T2 (startup
    reconciliation). The T1 spec MUST include a comment / docstring
    on `_current_claim_id` pointing at T2 as the recovery path for
    this case.
  - AC-T1-7 — No per-call timeout is added to the unclaim invocation;
    the SDK's transport-level timeout (the existing default the
    VtfWorkSource uses for all other calls) applies. Rationale: the
    outer k8s `terminationGracePeriodSeconds` is the binding ceiling,
    and stacking an inner timeout adds failure modes without
    materially improving worst-case outcomes.
---

# Spec

## Files touched

- `src/controller/controller.py` — state tracking + finally-block
  unclaim call.
- `src/controller/worksources/protocol.py` — add `unclaim` to the
  `WorkSource` Protocol.
- `src/controller/worksources/vtf.py` — implement `unclaim` against
  `self._client.tasks.unclaim` (which exists on `AsyncTaskManager`
  after T0 merges).
- `tests/test_controller.py` — new test + mock-method update.
- `pyproject.toml` — bump the pinned `vtf-sdk-python` commit hash to
  the T0 (`t_cHx0UU`) merge commit. This is part of T1's PR.

No other files. Specifically: **do not** modify
`src/controller/__main__.py`, the heartbeat module, the harness
invoker, the gate runner, the task data model, or anything in
`vtf-sdk-python` (T0 owns the SDK additions; T1 only consumes them
via the version pin-bump in `pyproject.toml`).

## controller.py changes

### `__init__` (around line 34-46)

Add one line, after the existing `self._agent_info = None`:

```python
self._current_claim_id: str | None = None
# ID of the task currently in-flight (claim succeeded but not yet
# reached terminal status). Set in _poll_and_execute, cleared on
# complete/fail/error. Used by the shutdown path in run() to release
# the claim before exit (vafi#4). Mid-execute SIGTERM is not covered
# by this field — recovery for that case is T2 startup reconciliation.
```

### `_poll_and_execute` (around lines 150-220)

Wrap the post-claim block in a try/finally so the field is always
cleared. Set the field immediately after the successful claim.

The existing structure:

```python
# Claim the task
logger.info(f"Claiming task {task.id}")
claimed_task = await self.work_source.claim(task.id, self._agent_info.id)
logger.info(f"Claimed task {claimed_task.id}: {claimed_task.title}")

# ... rework counting, log, execute, complete/fail ...
```

becomes:

```python
# Claim the task
logger.info(f"Claiming task {task.id}")
claimed_task = await self.work_source.claim(task.id, self._agent_info.id)
logger.info(f"Claimed task {claimed_task.id}: {claimed_task.title}")
self._current_claim_id = claimed_task.id
try:
    # ... rework counting, log, execute, complete/fail ...
finally:
    self._current_claim_id = None
```

The exception handler at the bottom of `_poll_and_execute` (the
`except Exception as e` that calls `self.work_source.fail(...)`)
moves inside the `try`, so `_current_claim_id` is still set when
`fail` is called. The `finally` clears it after.

Spec-author note: the existing `except Exception` block at lines
215-221 already handles the case where `claim` itself raises (in
which case `claimed_task` doesn't exist and `_current_claim_id`
was never set, which is correct). Keep that block where it is;
only wrap the post-claim section.

### `run` (around lines 52-138)

Extend the `finally` block (currently lines 120-137). Today it:

1. Cancels and awaits `agent_hb_task`.
2. Calls `set_agent_offline` (best-effort, logs on failure).
3. Logs "Shutting down controller".

Add a new step BEFORE step 2 (after the heartbeat cancel, before
set_agent_offline):

```python
# Release any in-flight claim (vafi#4 graceful-shutdown fix)
if self._current_claim_id is not None:
    try:
        await self.work_source.unclaim(self._current_claim_id)
        logger.info(
            f"Released in-flight claim {self._current_claim_id} on shutdown"
        )
    except Exception as e:
        logger.error(
            f"Failed to unclaim {self._current_claim_id} on shutdown: {e}",
            exc_info=True,
        )
    finally:
        self._current_claim_id = None
```

Ordering rationale: heartbeat cancel first (stop pretending to be
alive); then unclaim (release while still authenticated and the
agent is technically still online server-side); then
set_agent_offline (record we're done). If we did set_agent_offline
before unclaim, the server might apply policy that prevents an
offline agent from mutating claims.

## protocol.py changes

Add the method to the `WorkSource` Protocol, placed adjacent to
`claim` for locality:

```python
async def unclaim(self, task_id: str) -> None:
    """Release a previously-claimed task back to `todo`.

    Used by the shutdown path to surrender an in-flight claim
    cleanly before exit. State machine: doing -> todo. Server emits
    a dedicated `unclaimed` event. Authorization: the calling
    agent must be the current claimer.

    Args:
        task_id: Task ID to release.

    Raises:
        VtfConflictError: If the task is not in `doing` status, or
            is claimed by a different agent.
    """
    ...
```

## vtf.py changes

Implement the new method, mirroring the existing `claim` shape:

```python
async def unclaim(self, task_id: str) -> None:
    """Release a claim back to todo. See protocol docstring."""
    await self._client.tasks.unclaim(task_id)
```

Place it immediately after the existing `claim` method (around
line 76) for locality. No additional logging at this layer —
the controller logs the result, and the SDK transport logs the
HTTP call. Avoid duplicate noise.

## test_controller.py changes

Add a method to the existing `MockWorkSource` class
(`tests/test_controller.py:15`):

```python
async def unclaim(self, task_id: str) -> None:
    """Record the unclaim call for inspection by tests."""
    self.unclaim_calls.append(task_id)
```

And initialize `self.unclaim_calls: list[str] = []` in its
`__init__` (or wherever the mock's other call-tracking attrs are
initialized — match the existing pattern).

Add one new test inside `class TestController`, modeled on
`test_shutdown_signal_handling` (line 257) and
`test_marks_agent_offline_on_shutdown` (line 279):

```python
async def test_unclaims_in_flight_claim_on_shutdown(
    self, mock_work_source, test_config, sample_agent, sample_task
):
    """SIGTERM during a held claim releases it before agent goes offline.

    Verifies vafi#4 graceful-shutdown fix. The controller must call
    work_source.unclaim() before work_source.set_agent_offline() so
    that the release happens while still authenticated.
    """
    # Arrange: register succeeds, claim succeeds, execute blocks
    # indefinitely until we set the shutdown flag.
    mock_work_source.register_response = sample_agent
    mock_work_source.poll_response = sample_task
    mock_work_source.claim_response = sample_task
    execute_blocker = asyncio.Event()
    # ... (mock execute hook that awaits execute_blocker)

    controller = Controller(mock_work_source, test_config)

    # Act: start the controller, wait until the claim has happened,
    # then signal shutdown.
    run_task = asyncio.create_task(controller.run())
    await _wait_until(lambda: mock_work_source.claim_calls, timeout=2.0)
    controller._shutdown.set()
    execute_blocker.set()  # release the blocked execute path
    await asyncio.wait_for(run_task, timeout=5.0)

    # Assert: unclaim called exactly once with the right task ID,
    # and before set_agent_offline.
    assert mock_work_source.unclaim_calls == [sample_task.id]
    # Ordering — captured via a single ops log on the mock:
    assert mock_work_source.ops.index("unclaim") < \
           mock_work_source.ops.index("set_agent_offline")
```

Spec-author note: the exact mock-introspection shape (`ops` list,
`claim_calls`, `_wait_until` helper) is illustrative — match the
mock's existing conventions. If the existing mock doesn't track
call ordering across method types, add the minimum needed
(a single `self.ops: list[str] = []` that each tracked method
appends to is the lightest-weight pattern).

# Constraints

- MUST NOT change the Task data model or any vtaskforge-side code
  (Phase 1 single-repo constraint per
  `implementation-roadmap-PLAN.md` §"Phase 1").
- MUST NOT block process exit past `terminationGracePeriodSeconds`.
  This is enforced by relying on the SDK's transport timeout and
  the fact that the only awaited call added to the shutdown path
  is a single HTTP POST.
- MUST work whether or not the agent heartbeat loop is currently
  active — the unclaim call goes to vtaskforge directly via the
  SDK, not through the heartbeat path.
- MUST NOT introduce a new `WorkSource` implementation; modify only
  the existing `VtfWorkSource`. (If/when a second implementation is
  added, that contributor implements `unclaim` per the protocol.)
- MUST NOT modify `src/controller/__main__.py`. The existing entry
  point already wires the controller correctly.

# Out of scope

- Mid-`execute` SIGTERM cancellation (see AC-T1-6). Delegated to T2
  startup reconciliation.
- Adding a SIGINT-specific path. SIGTERM and SIGINT both already set
  the same `_shutdown` event; the unclaim path treats them
  identically.
- Differentiating "sigterm-graceful-release" from other unclaim
  reasons at the server (the existing unclaim endpoint has no
  reason payload; pod logs differentiate them).
- Changing the `terminationGracePeriodSeconds` default or
  documenting one — that's a deployment-side concern (helm
  chart / argocd manifest), not a controller change.

# References

- workgraph.md (this directory) AC-2 — the DAG-level "must release
  within budget" criterion that T1 operationalizes.
- plan.md (this directory) §"T1 — SIGTERM graceful release" — the
  prose approach this spec implements.
- `viloforge-platform/docs/vafi-runtime-DESIGN.md` §"Phase 5 —
  Draining" — the architectural design for this behavior, including
  the heartbeat-timeout server-side backstop referenced in AC-T1-3.
- `vafi/src/controller/controller.py:34-46` — `__init__` (where
  `_current_claim_id` is initialized).
- `vafi/src/controller/controller.py:120-137` — `run`'s existing
  `finally` block (where the unclaim step is inserted).
- `vafi/src/controller/controller.py:150-220` — `_poll_and_execute`
  (where claim tracking starts/ends).
- `vafi/src/controller/controller.py:532-539` —
  `_setup_signal_handlers` (unchanged; provides the existing
  cooperative-shutdown plumbing this fix relies on).
- `vafi/src/controller/worksources/protocol.py` — `WorkSource`
  protocol.
- `vafi/src/controller/worksources/vtf.py` — `VtfWorkSource`
  implementation.
- `vafi/tests/test_controller.py:15` — `MockWorkSource` (test
  double extended).
- `vafi/tests/test_controller.py:257`, `:279` —
  `test_shutdown_signal_handling`, `test_marks_agent_offline_on_shutdown`
  (model for the new test).
- vtaskforge `tasks/views.py:269` — `unclaim` endpoint (verified to
  exist server-side).
- vtaskforge `vtf-sdk-python/vtf_sdk/managers.py:86` — sync
  `TaskManager.unclaim` (already exists; template for the async
  version T0 adds).
- vtaskforge `vtf-sdk-python/vtf_sdk/async_client.py` —
  `AsyncTaskManager` (where T0 adds `unclaim`; the call site T1
  depends on).
- T0 task: `00-add-sdk-methods.md` — sibling task that produces the
  `AsyncTaskManager.unclaim` method T1 calls.
