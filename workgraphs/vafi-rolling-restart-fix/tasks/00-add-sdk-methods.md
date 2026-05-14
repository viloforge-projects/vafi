---
id: t_cHx0UU
slug: add-sdk-methods
title: Expose unclaim + agents.tasks endpoints in vtf-sdk-python's async client
workgraph: vafi-rolling-restart-fix
order: 0
required_tags:
  - executor
depends_on: []
target_repo: vtaskforge
agent_model: claude-opus-4-7
judge: true
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T0-1 — `AsyncTaskManager.unclaim(id: str) -> Task` is added to
    `vtf-sdk-python/vtf_sdk/async_client.py`. Implementation mirrors
    the existing sync `TaskManager.unclaim` in `managers.py:86` —
    `await self._transport.post(f"/v2/tasks/{id}/unclaim/")`,
    returning `Task.model_validate(data)`. Placed in
    `AsyncTaskManager` adjacent to `claim` (alphabetical not required;
    co-locate with `claim` for readability).
  - AC-T0-2 — `AgentManager.tasks(agent_id: str, *, status: str | None = None) -> PagedResult[Task]`
    is added to `vtf-sdk-python/vtf_sdk/managers.py`. Implementation
    calls `self._transport.get(f"/v2/agents/{agent_id}/tasks/", params=...)`
    where `params` is `{"status": status}` when status is provided,
    `None` otherwise. Response parsed via `_parse_paged(data, Task)`.
    Mirrors the existing `AgentManager.list` shape (managers.py:263).
  - AC-T0-3 — `AsyncAgentManager.tasks(agent_id: str, *, status: str | None = None) -> PagedResult[Task]`
    is added to `vtf-sdk-python/vtf_sdk/async_client.py`. Async
    version of AC-T0-2 — same params, same response shape,
    `await self._transport.get(...)`.
  - AC-T0-4 — Unit tests cover the three new methods.
    `tests/test_managers_read.py` (or whichever existing test file
    covers `AgentManager.list`) gains a test for
    `AgentManager.tasks(agent_id, status="doing")` verifying the
    correct URL + params + response parsing. `tests/test_async_client.py`
    gains tests for both `AsyncTaskManager.unclaim` and
    `AsyncAgentManager.tasks`, following the existing async-test
    pattern (mock transport, assert HTTP call shape + return value).
    Tests pass under the existing `pytest` invocation in the repo's
    CI workflow.
  - AC-T0-5 — No vtaskforge server-side code is modified — not
    `src/tasks/views.py`, not `src/agents/views.py`, not any
    Django model, serializer, URL config, or migration. Only files
    under `vtf-sdk-python/` change. (The endpoints these methods
    wrap already exist server-side; this task only exposes them
    in the async SDK surface.)
  - AC-T0-6 — vafi's `pyproject.toml` is updated to pin
    `vtf-sdk-python` to the merge-commit hash of this PR. The
    pin-bump is committed in the **vafi repo as part of T1's PR**
    (since T1 is the first vafi-side task that requires the new
    SDK surface), NOT in a separate vafi commit attached to T0.
    Rationale: T0's deliverable is "vtaskforge merge commit
    exists"; the vafi-side consumption of that commit is T1's
    business.
  - AC-T0-7 — The vtaskforge PR commit message clearly states the
    workgraph + task IDs (`vafi-rolling-restart-fix / t_cHx0UU`)
    and links the GitHub issue (`viloforge/vafi#4`) so a future
    reader can trace this SDK alignment back to the workgraph
    that required it.
---

# Spec

## Files touched

In the `vtaskforge` repo:

- `vtf-sdk-python/vtf_sdk/managers.py` — add `AgentManager.tasks(...)`.
- `vtf-sdk-python/vtf_sdk/async_client.py` — add
  `AsyncTaskManager.unclaim(...)` and `AsyncAgentManager.tasks(...)`.
- `vtf-sdk-python/tests/test_managers_read.py` (or sibling) — test
  for the new sync `AgentManager.tasks`.
- `vtf-sdk-python/tests/test_async_client.py` — tests for the two
  new async methods.

NOT touched: anything in `vtaskforge/src/` (server-side Django
project) — see AC-T0-5.

## Implementation sketch

### `managers.py` — `AgentManager.tasks`

After the existing `update` method (managers.py:282-284), add:

```python
def tasks(self, agent_id: str, *,
          status: str | None = None,
          page_size: int = 50) -> PagedResult[Task]:
    """List tasks claimed by this agent's user account.

    Wraps `GET /v2/agents/<agent_id>/tasks/`. Filters by status
    when provided.

    Note: the server filters by `claimed_by=agent.user`, not by
    agent-instance ID. Safe for single-active-replica deployments
    only; multi-replica fan-out under a shared agent ID requires
    a different server-side filter (out of scope here).

    Args:
        agent_id: Agent ID.
        status: Optional task status filter (e.g., "doing").
        page_size: Page size for the response.

    Returns:
        Paged Task results.
    """
    params: dict = {"page_size": page_size}
    if status is not None:
        params["status"] = status
    data = self._transport.get(
        f"/v2/agents/{agent_id}/tasks/",
        params=params,
    )
    return _parse_paged(data, Task)
```

Import `Task` and `_parse_paged` if not already imported in
this module's `AgentManager` block — they're already in use by
`TaskManager` so the imports should be in scope.

### `async_client.py` — `AsyncTaskManager.unclaim`

After the existing `claim` method (line 59-61), add:

```python
async def unclaim(self, id: str) -> Task:
    """Release a claimed task back to todo.

    Wraps `POST /v2/tasks/<id>/unclaim/`. State machine:
    `doing -> todo`. Clears `claimed_by`, `claimed_at`,
    `claim_expires_at`. Emits an `unclaimed` event.

    Authorization: the calling agent must be the current claimer.

    Raises:
        VtfConflictError: If the task is not in `doing`, or claimed
            by a different agent.
    """
    data = await self._transport.post(f"/v2/tasks/{id}/unclaim/")
    return Task.model_validate(data)
```

### `async_client.py` — `AsyncAgentManager.tasks`

After the existing `update_status` method in `AsyncAgentManager`
(line 133-135), add:

```python
async def tasks(self, agent_id: str, *,
                status: str | None = None,
                page_size: int = 50) -> PagedResult[Task]:
    """Async version of AgentManager.tasks. See sync docstring."""
    params: dict = {"page_size": page_size}
    if status is not None:
        params["status"] = status
    data = await self._transport.get(
        f"/v2/agents/{agent_id}/tasks/",
        params=params,
    )
    return _parse_paged(data, Task)
```

Mirror imports as needed (`Task`, `_parse_paged`, `PagedResult`).

### Tests

Each new method gets a test that:

1. Constructs a client with a mock transport.
2. Calls the method.
3. Asserts the correct HTTP verb + URL + params.
4. Asserts the parsed response matches.

The existing test patterns in `tests/test_managers_*.py` and
`tests/test_async_client.py` show the exact shape — spec-author
phase here defers to the executor to match the existing repo
conventions for fixture wiring and mock-transport setup.

# Constraints

- MUST NOT modify any vtaskforge server-side code (Django project
  under `src/`). The endpoints already exist; this PR only exposes
  them via SDK methods.
- MUST NOT change existing SDK method signatures or behavior. The
  three new methods are pure additions.
- MUST add tests; merging without tests is a methodology violation
  (P3 in architect-retro — "verify the FULL call chain").
- MUST NOT remove or modify the sync `TaskManager.unclaim`
  (managers.py:86); the async addition mirrors it, doesn't replace
  it.
- The PR title and commit message MUST cite this workgraph+task
  (`vafi-rolling-restart-fix / t_cHx0UU`) and link `viloforge/vafi#4`
  per AC-T0-7.

# Out of scope

- Adding `AsyncTaskManager.reset` (the sync `TaskManager.reset`
  exists but isn't needed by this workgraph; mirroring it to async
  is a different methodology question — admin endpoints shouldn't
  necessarily be in the async client used by agent code).
- Any `/v2/tasks/<id>/release/` endpoint or SDK method (rejected in
  plan.md Alt B).
- Server-side filter-by-agent-instance-ID (the safety constraint
  flagged in workgraph.md "Out of scope" — separate workgraph if
  multi-replica deployments are ever needed).
- vafi's `pyproject.toml` pin-bump — that's in T1 per AC-T0-6.

# References

- workgraph.md (this directory) — DAG topology with T0 → T1, T2.
- plan.md (this directory) §"T0 — Add missing async SDK methods" —
  prose approach.
- architect-retro.md P12 — the methodology gap that caused T0 to
  be missed in architect phase.
- `vtaskforge/vtf-sdk-python/vtf_sdk/managers.py:86` — existing
  sync `TaskManager.unclaim` (template for async version).
- `vtaskforge/vtf-sdk-python/vtf_sdk/managers.py:263` — existing
  `AgentManager.list` (template for `tasks` shape).
- `vtaskforge/vtf-sdk-python/vtf_sdk/async_client.py:59` — existing
  `AsyncTaskManager.claim` (placement reference for `unclaim`).
- `vtaskforge/vtf-sdk-python/vtf_sdk/async_client.py:108-135` —
  `AsyncAgentManager` (where `tasks` is added).
- `vtaskforge/src/tasks/views.py:269` — server-side `unclaim`
  endpoint (already exists).
- `vtaskforge/src/agents/views.py:85` — server-side
  `agents/<id>/tasks/` endpoint (already exists).
