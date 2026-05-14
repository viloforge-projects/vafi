---
id: t_UlglCJ
slug: integration-test-rolling-restart
title: Integration test — induced rolling restart leaves no stranded claims
workgraph: vafi-rolling-restart-fix
order: 3
required_tags:
  - executor
depends_on:
  - t_4ZZ60T            # T1 — needs SIGTERM unclaim path in place
  - t_QrwliD            # T2 — needs startup-reconciliation path in place
target_repo: viloforge/vafi
agent_model: claude-opus-4-7
judge: true
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T3-1 — A new integration test exists at
    `tests/integration/test_rolling_restart.py`, marked
    `@pytest.mark.integration` (matching the existing convention
    in `tests/integration/test_session_continuity.py`). The test
    is NOT collected by default; it runs only when explicitly
    invoked via the new Makefile target.
  - AC-T3-2 — A new Makefile target `integration-test-rolling-restart`
    SHALL:
    (a) verify `kind` and `kubectl` are installed (fail fast with
        install hints if not);
    (b) create a kind cluster named `vafi-int` (or reuse an
        existing one if `KEEP_CLUSTER=1` env var is set);
    (c) load the just-built controller image into the kind cluster;
    (d) install a minimal vtaskforge + vafi deployment subset
        (the test fixture, NOT the full charts/vafi helm chart);
    (e) run `pytest tests/integration/test_rolling_restart.py -m integration -v`;
    (f) tear down the kind cluster on success or failure (unless
        `KEEP_CLUSTER=1`).
  - AC-T3-3 — The test SHALL be environment-agnostic: it uses
    `KUBECONFIG` to locate the cluster and uses standard `kubectl`
    + vtaskforge HTTP calls. No assumption that the cluster is
    kind specifically — the same test runs unchanged in any
    Kubernetes cluster, which means a future CI workflow can
    invoke it without test-code changes.
  - AC-T3-4 — A test-only harness-invoker mode SHALL be added:
    when the env var `VF_TEST_HOLD_CLAIM_SECONDS` is set to a
    positive integer N, the harness invoker (`src/controller/invoker.py`)
    skips real harness invocation and instead `asyncio.sleep(N)`s
    while holding the claim. This makes the test deterministic
    without requiring a real long-running harness invocation.
    The env var name `VF_TEST_HOLD_CLAIM_SECONDS` is reserved for
    test fixtures — production deployments do NOT set it.
  - AC-T3-5 — **Scenario A (T1 SIGTERM path)** in the test:
    submits a task with `VF_TEST_HOLD_CLAIM_SECONDS=120`, waits
    for the executor pod to claim it (poll until task status is
    `doing`), then runs
    `kubectl rollout restart deployment/executor-pi`. After
    `terminationGracePeriodSeconds + 30s`, the test polls
    `GET /v2/tasks/<task_id>/` until `status == "todo"` (timeout
    60s after the wait). MUST pass.
  - AC-T3-6 — **Scenario B (T2 startup-reconciliation path)** in
    the test: same setup as Scenario A, but instead of
    `rollout restart` the test issues
    `kubectl delete pod <pod_name> --grace-period=0` (simulates
    hard kill / OOM where SIGTERM is not delivered to the
    controller's signal handler). Wait for the replacement pod to
    come up (poll until a new pod is `Ready`). Poll
    `GET /v2/tasks/<task_id>/` until `status == "todo"`. MUST
    pass — this validates that T2 recovers the claim T1 couldn't.
  - AC-T3-7 — The test SHALL clean up after itself: delete the
    task it created, regardless of pass/fail. Cleanup runs in a
    pytest fixture's teardown.
  - AC-T3-8 — A README section or `tests/integration/README.md`
    SHALL document how to run the integration test locally:
    prereqs (kind + kubectl + docker), command
    (`make integration-test-rolling-restart`), expected runtime
    (~3-5 min for kind spinup + test), and how to debug
    (`KEEP_CLUSTER=1 make integration-test-rolling-restart` to
    inspect the cluster after the run).
  - AC-T3-9 — CI wiring is OUT OF SCOPE. The Makefile target is
    the operator-facing handle today; a follow-up `kind: infrastructure`
    workgraph will wire `make integration-test-rolling-restart`
    into GitHub Actions. The architect-retro W4 walk-back
    documents this deferral.
---

# Spec

## Files touched

- `Makefile` — new target `integration-test-rolling-restart` +
  any helper targets (e.g., `_kind-up`, `_kind-down`,
  `_load-images`) the main target composes from.
- `tests/integration/test_rolling_restart.py` — new test (NEW
  FILE).
- `tests/integration/fixtures/vafi-int-minimal.yaml` — minimal
  k8s manifest (vtaskforge + vafi executor) for the kind
  test fixture (NEW FILE; or split across multiple files
  per usual yaml hygiene).
- `tests/integration/README.md` — runbook (NEW FILE, or update
  if it exists).
- `src/controller/invoker.py` — recognize the
  `VF_TEST_HOLD_CLAIM_SECONDS` env var and short-circuit to
  `asyncio.sleep(N)` when set.
- (Possibly) `scripts/integration/setup-kind.sh` — shell script
  the Makefile invokes; depends on how the project structures
  Makefile helpers. Match existing repo conventions.

NOT touched:
- T1's files (`controller.py` register / shutdown logic).
- T2's files (`_reconcile_orphaned_claims` method).
- `charts/vafi/` — the helm chart for prod deployment is NOT the
  test fixture. The test fixture is a minimal subset specifically
  for integration testing.

## Test design

### Fixture: minimal vtaskforge + vafi on kind

`tests/integration/fixtures/vafi-int-minimal.yaml` deploys:

- A vtaskforge pod (using a published image; pull from
  whatever registry vtaskforge publishes to — see existing
  references in helm chart's image values).
- A SQLite-backed vtaskforge config (no Postgres; minimal).
- An `executor-pi` Deployment with `replicas=1`, the locally-built
  controller image (loaded via `kind load`), and
  `terminationGracePeriodSeconds=30`.
- Enough RBAC / Service / Secret to make vtaskforge reachable
  from the executor pod and the test harness.

The fixture is SEPARATE from the production helm chart because:
(a) the prod chart pulls live config from external secret
managers, which won't work in a transient kind cluster;
(b) the prod chart is N services; the test fixture is just
enough to exercise the controller restart behavior.

### Test structure

```python
import pytest
import subprocess
import time
import requests
import os

pytestmark = pytest.mark.integration

VTF_URL = os.environ.get("VTF_URL", "http://localhost:30080")  # NodePort
VTF_TOKEN = os.environ.get("VTF_TOKEN", "...")  # supplied by setup script
NAMESPACE = "vafi-int"


@pytest.fixture
def claim_holding_task():
    """Create a task that holds the claim for 120s (via test env flag).

    Yields the task ID; teardown deletes the task.
    """
    task_id = _create_task_via_vtf_api(hold_seconds=120)
    yield task_id
    _delete_task_via_vtf_api(task_id)


def test_sigterm_path_releases_claim(claim_holding_task):
    """T1: SIGTERM during a held claim → task back to todo."""
    task_id = claim_holding_task
    _wait_for_task_status(task_id, "doing", timeout=30)

    pod_name = _get_executor_pod_name()
    subprocess.run(
        ["kubectl", "rollout", "restart", "deployment/executor-pi",
         "-n", NAMESPACE],
        check=True,
    )

    # After terminationGracePeriodSeconds + buffer, task must be back in todo.
    _wait_for_task_status(task_id, "todo", timeout=60)


def test_hard_kill_path_releases_claim(claim_holding_task):
    """T2: hard kill (no SIGTERM) → next pod's startup-recon releases the claim."""
    task_id = claim_holding_task
    _wait_for_task_status(task_id, "doing", timeout=30)

    pod_name = _get_executor_pod_name()
    subprocess.run(
        ["kubectl", "delete", "pod", pod_name, "-n", NAMESPACE,
         "--grace-period=0", "--force"],
        check=True,
    )

    # Wait for new pod to come up.
    _wait_for_pod_ready(deployment="executor-pi", timeout=60)

    # T2 startup reconciliation should release the orphaned claim.
    _wait_for_task_status(task_id, "todo", timeout=60)
```

The helper functions (`_create_task_via_vtf_api`,
`_wait_for_task_status`, `_get_executor_pod_name`,
`_wait_for_pod_ready`) implement the obvious behaviors. Match
the existing helper-function pattern in `test_session_continuity.py`
if it offers a useful template; otherwise inline them in this
test module for now (no premature abstraction).

### Harness invoker test mode

In `src/controller/invoker.py`, add at the very top of the
harness-invocation method (whichever method is the entry point —
the spec-author defers to the executor to locate; the
`HarnessInvoker` class is at `controller.py:45` reference):

```python
hold_seconds = os.environ.get("VF_TEST_HOLD_CLAIM_SECONDS")
if hold_seconds:
    try:
        n = int(hold_seconds)
        if n > 0:
            logger.info(
                f"VF_TEST_HOLD_CLAIM_SECONDS={n} — test-only mode, "
                f"sleeping instead of invoking harness"
            )
            await asyncio.sleep(n)
            # Return whatever ExecutionResult shape the caller expects;
            # match the existing success-path shape.
            return ExecutionResult(success=True, completion_report="test-mode sleep complete")
    except ValueError:
        logger.warning(
            f"VF_TEST_HOLD_CLAIM_SECONDS={hold_seconds!r} is not a valid int; ignoring"
        )
```

The env var name MUST start with `VF_TEST_` so it's
self-documenting as test-only; production manifests MUST NOT
set it.

### Makefile target

```make
.PHONY: integration-test-rolling-restart
integration-test-rolling-restart: _check-kind _kind-up _load-images _deploy-fixture _run-int-test _kind-down

.PHONY: _check-kind
_check-kind:
	@command -v kind >/dev/null || { echo "kind not installed; see https://kind.sigs.k8s.io/docs/user/quick-start/"; exit 1; }
	@command -v kubectl >/dev/null || { echo "kubectl not installed"; exit 1; }

.PHONY: _kind-up
_kind-up:
	@if [ "$$KEEP_CLUSTER" = "1" ] && kind get clusters | grep -q vafi-int; then \
	  echo "Reusing existing vafi-int cluster"; \
	else \
	  kind create cluster --name vafi-int; \
	fi

.PHONY: _load-images
_load-images:
	docker build -t vafi-controller:int .
	kind load docker-image vafi-controller:int --name vafi-int

.PHONY: _deploy-fixture
_deploy-fixture:
	kubectl apply -f tests/integration/fixtures/vafi-int-minimal.yaml
	kubectl -n vafi-int wait --for=condition=available deployment --all --timeout=120s

.PHONY: _run-int-test
_run-int-test:
	pytest tests/integration/test_rolling_restart.py -m integration -v

.PHONY: _kind-down
_kind-down:
	@if [ "$$KEEP_CLUSTER" = "1" ]; then \
	  echo "KEEP_CLUSTER=1; leaving cluster up. Tear down with: kind delete cluster --name vafi-int"; \
	else \
	  kind delete cluster --name vafi-int; \
	fi
```

The exact Makefile shape matches the repo's existing conventions
— check whether the Makefile uses spaces vs tabs, $$ vs $,
existing target naming conventions, etc. The structure above is
illustrative.

`_kind-down` MUST run on test failure (the `&&` chain in the main
target won't do that). Use a `trap`-style shell wrapper or
restructure as a single shell command with `trap` to ensure
teardown always fires. The spec-author defers the precise shape
to the executor — the requirement is "teardown always runs unless
`KEEP_CLUSTER=1`."

# Constraints

- MUST NOT modify T1 or T2 source code. T3 is purely additive.
- MUST NOT make the integration test part of the default `pytest`
  invocation. It MUST require the `-m integration` marker or
  explicit invocation via the Makefile target.
- MUST NOT bake any CI-specific assumptions into the test or
  Makefile target. Both must work on an operator's local
  machine with kind + kubectl + docker.
- MUST NOT pollute the operator's existing kind clusters. The
  test cluster is named `vafi-int` specifically; do not use
  `kind` (the default cluster name).
- The `VF_TEST_HOLD_CLAIM_SECONDS` flag MUST NOT affect
  production behavior — the env var is read once at harness
  invocation; if absent, the existing real-harness path runs
  unchanged.
- The minimal test fixture MUST NOT pull from any private
  registries the operator's local docker doesn't have
  credentials for. Use publicly-available images for vtaskforge
  (or document a `docker pull` prereq if a private image must
  be used; spec-author defers to executor to confirm vtaskforge
  publishes a public image).

# Out of scope

- Wiring `make integration-test-rolling-restart` into GitHub
  Actions or any other CI system (W4 walk-back; follow-up
  `kind: infrastructure` workgraph per Q6).
- Running the test against a non-kind ephemeral cluster (k3d,
  minikube, etc.). The test logic is environment-agnostic, but
  the Makefile target hard-codes kind specifically. If/when
  another local-cluster tool is preferred, that's a small
  Makefile change in a follow-up.
- Performance testing of the unclaim path (e.g., 100 stranded
  claims at once). Realistic max is 0-1; perf isn't the concern.
- Testing against multiple vafi controller replicas (the
  multi-replica safety constraint is documented as out-of-scope
  in workgraph.md and runtime DESIGN).
- Resumable harness sessions (referenced in issue body as future
  work).

# References

- workgraph.md (this directory) AC-4 — DAG-level "verified by
  ephemeral kind/k3d integration test" criterion T3 operationalizes.
- plan.md (this directory) §"T3 — Integration test" — prose
  approach with the two scenarios.
- 01-impl-sigterm-graceful-release.md (T1) — Scenario A relies on
  T1's SIGTERM unclaim path.
- 02-impl-startup-reconciliation.md (T2) — Scenario B relies on
  T2's startup-reconciliation path.
- architect-retro.md W4 + P13 + Q6 — the CI-wiring walk-back, the
  methodology pattern it produced, and the follow-up workgraph
  question it spawns.
- `vafi/tests/integration/test_session_continuity.py` — existing
  integration test convention (`@pytest.mark.integration`,
  manual run, kubectl + HTTP).
- `vafi/Makefile` — existing make conventions to mirror.
- `vafi/src/controller/invoker.py` — where the
  `VF_TEST_HOLD_CLAIM_SECONDS` flag is read.
- `vafi/charts/vafi/templates/executor-pi-deployment.yaml` —
  reference for what the minimal test-fixture deployment looks
  like (subset of, not copy of).
- kind: https://kind.sigs.k8s.io/docs/user/quick-start/
