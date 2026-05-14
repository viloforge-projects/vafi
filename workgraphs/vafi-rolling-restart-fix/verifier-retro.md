---
stage: verifier
workgraph: vafi-rolling-restart-fix
phase: methodology-extraction
authored_by: operator:lonvilo
co_authors:
  - claude-opus-4-7
created_at: 2026-05-14T00:00:00Z
session_duration_minutes: ~30
session_kind: interactive
tasks_verified:
  - t_cHx0UU (T0)
  - t_4ZZ60T (T1)
  - t_QrwliD (T2)
  - t_UlglCJ (T3)
verdict: ready
---

# Verifier-phase retro — vafi-rolling-restart-fix

## Status

Methodology-extraction scaffolding (architect-retro X5 corrected
framing). Co-located in workgraph dir; archived once
`vtf-methodologies/verifier/*.md` stabilizes.

## Verdict

**Workgraph status: `specced → ready`**. All four task specs pass
the verify checklist (after two inline patches applied during this
phase). The workgraph is now ready to enter execution.

## Verify checklist results

| Check | Result | Notes |
|---|---|---|
| `[NEEDS CLARIFICATION]` markers | ✓ zero | Only descriptive references in architect-retro |
| DAG acyclic + connected | ✓ | T0→T1, T0→T2, {T1,T2}→T3 |
| `depends_on` IDs resolve | ✓ | All 4 IDs reachable in this workgraph dir |
| `target_repo` resolves | ✓ | vtaskforge ✓, viloforge/vafi ✓ |
| AC verifiability (judge-gradeable) | ✓ | All ACs independently testable |
| file:line citations | ✓ | Spot-checked 4 of ~20; all exact matches |
| OOS items don't contradict ACs | ✓ | None found |
| Methodology pinning consistency | n/a | No methodologies yet — Phase 1 |
| **Mock-double idiom match** | ✗ **found 1, patched inline** | See V1 below |
| **Cross-task AC agreement** | ✗ **found 1, patched inline** | See V2 below |

## Findings + patches applied during verify

### F1 — Mock idiom mismatch (T1, T2 specs)

**Found.** Both T1's AC-T1-5 and T2's AC-T2-6/7/8 used custom
state-tracking idioms (`unclaim_calls.append(...)`,
`list_active_claims_response = [...]`, `unclaim_errors = {...}`)
that don't match the existing `MockWorkSource` convention. The
existing class uses `unittest.mock.AsyncMock()` for every protocol
method (`tests/test_controller.py:15-32`) — no manual list-tracking
attributes anywhere.

**Patched.** T1 mock section rewritten to use AsyncMock idiom:
- `self.unclaim = AsyncMock()` in `__init__`.
- `mock.unclaim.assert_awaited_once_with(task_id)` for single-call
  assertions.
- `mock.Mock()` + `attach_mock(...)` parent recorder for cross-
  method ordering.

T2 mock section rewritten similarly:
- `mock.list_active_claims.return_value = [...]` for happy-path.
- `mock.unclaim.side_effect = [None, RuntimeError(...), None]` for
  per-task error scenarios.
- `mock.method.await_args_list` for ordered call inspection.

Methodology finding: **V1** (below).

### F2 — Cross-task AC ordering ambiguity (T0/T1 pin-bump)

**Found.** AC-T0-6 prescribed "the pin-bump is committed in T1's
PR" — opinionated about execution order. But T1 and T2 are both
T0-dependent and independent of each other; either can land first.
If T2 lands first, the pin-bump belongs in T2's PR (otherwise T2
can't import the new SDK methods).

**Patched.** AC-T0-6 reworded to "whichever T1/T2 PR lands first
carries the pin-bump; the second PR inherits the pin." AC-T1-4
matched language to be consistent. AC-T2 frontmatter does not need
patching — it doesn't currently prescribe pin-bump ordering.

Methodology finding: **V2** (below).

## Verifier-specific methodology candidates

### V1. Verify mock-double idioms against the existing test scaffold

**Maturity:** HIGH — produced F1 in this workgraph; preventable
defect from a spec-author who didn't open the existing
`MockWorkSource` before writing test sketches.

**Observed.** Spec-author retro SA4 named "mock-double extension
is part of the spec." Spec-author then violated SA4 by writing
T1/T2 mock sections that didn't match the existing test scaffold's
AsyncMock convention. The verifier caught it.

**Candidate rule.** For every task spec that introduces a new
protocol/interface method or modifies an existing test-double, the
verifier opens the actual test-double class in the target repo and
checks:

1. The mock convention in use (AsyncMock vs custom class vs
   manual tracking).
2. The expected initialization pattern (where new mock attributes
   are added).
3. The assertion style the existing tests use (AsyncMock
   `.assert_awaited_with(...)` vs manual `list ==` comparison).

If the spec doesn't match, patch the spec inline OR escalate to
spec-author rework. Don't pass the buck to executor — they'll
match whatever the spec says, even when the spec is wrong about
the local idiom.

**Why this is a verifier pattern, not just spec-author follow-up.**
Spec-author SA4 SHOULD catch this. In practice (this workgraph)
SA4 wasn't enforced rigorously. The verifier is the next gate;
if it doesn't catch the mock mismatch, the executor implements
against the wrong idiom and the judge has to choose between
grading "it matches the spec but is wrong code" or "it doesn't
match the spec but is right code" — both bad outcomes.

**Cross-link to SA4.** SA4 is the spec-author-side rule
("specify the mock"). V1 is the verifier-side rule ("verify the
specified mock matches reality"). Both are needed because
spec-author can write a confident-looking mock sketch without
having opened the test file.

### V2. Cross-task AC consistency when ACs reference shared artifacts

**Maturity:** HIGH — produced F2 in this workgraph.

**Observed.** AC-T0-6 referenced "T1's PR" — implying the
pin-bump goes there. But the DAG allows T1 and T2 to execute in
either order (both depend only on T0). When two ACs from
different tasks reference the same artifact (a PR, a commit hash,
a file path, an entity), they must either:

- Agree on the artifact's location/owner;
- OR be neutral about ordering;
- OR explicitly name the precedence rule.

**Candidate rule.** The verifier checks every cross-task AC
reference for consistency:

1. List all ACs that name another task by ID (`t_xxxxxx`) or by
   role ("T1's PR", "T2's mock", "the integration test").
2. For each, verify the named task's ACs agree (no conflicting
   prescriptions).
3. If a referenced artifact has multiple plausible owners (e.g.,
   pin-bump goes in "whichever PR lands first"), the AC text
   must explicitly handle the ordering or pick a canonical
   owner with rationale.

**Anti-pattern.** ACs that bake in an execution order the DAG
doesn't enforce. The DAG's `depends_on` topology IS the
execution-order contract; ACs that say "X happens in Y's PR" must
either reflect the DAG ordering or commit to a specific order
with rationale.

### V3. Spot-check file:line citations cheaply catches drift

**Maturity:** MEDIUM (only spot-checked 4 of ~20 in this
workgraph; corroboration needed)

**Observed.** All 4 spot-checked file:line citations were exact
matches. Cost: ~30 seconds per check (`sed -n '<n>,<m>p' <file>`
or equivalent). Value: catches stale citations from refactors
that happened between spec-author and verify phases.

**Candidate rule.** Verifier spot-checks 4–6 file:line citations
across the workgraph, biased toward the most-referenced files
(by AC count). Doesn't need to be exhaustive; the goal is to
catch repo-drift (e.g., a function moved from line 81 to line 92
during a sibling PR). One miss is fine; multiple misses signal
the workgraph is stale and needs re-spec-authoring.

**For the autonomous verifier agent variant.** This is mechanical
and should run automatically: parse all `file_path:line` patterns,
read the cited lines, check they match the expected content
(function name, comment, etc.). Failures get logged as drift
findings (X3 from architect-retro).

## Cross-cutting findings

### VX1. Verifier is a low-cost, high-leverage gate

**Observed.** The two findings (F1, F2) took ~30 minutes total to
catch and patch. If they had landed at executor phase:
- F1: executor implements `unclaim_calls`-style mock, all tests
  pass but the mock looks wrong to anyone familiar with the
  existing scaffold. Judge sees the mismatch but doesn't know if
  it's a spec-vs-code drift or an executor improv. Rework loop
  through spec-author at minimum.
- F2: T1 and T2 PRs both try to update the pin-bump independently.
  Merge conflict, or one PR's bump silently overwritten. Operator
  intervention.

**Cross-cutting form.** Verifier-phase cost is bounded (a single
read-only pass over the workgraph dir). Verifier-phase value is
proportional to the count of defects it catches before they reach
executor. Phase 1 methodology should invest in making verifier
checks as mechanical as possible — they pay back disproportionately.

### VX2. Verifier should be the LAST gate where spec-vs-reality is checked

**Observed.** SA4 named mock-spec discipline as a spec-author
responsibility. SA1 named full-chain verification as a spec-author
responsibility. In practice, both got violated in this workgraph
and verifier caught them.

**Cross-cutting form.** Verifier's role is "last line of defense
for spec-vs-reality drift." The earlier stages (triage, architect,
spec-author) SHOULD catch their respective drifts (per their
methodology rules). When they don't, verifier MUST. By the time
the workgraph is `ready`, every spec should be defensible against:

- "Does this file exist where you say it does?" (file-path
  citations)
- "Does this method exist in the SDK?" (P12)
- "Does this test framework exist?" (P13)
- "Does this mock convention match the existing scaffold?" (V1)
- "Do cross-task ACs agree?" (V2)

If verifier can answer "yes" to all of those, the workgraph is
ready. If "no," patch inline or reject back to the relevant earlier
stage.

## Open questions for the next workgraph

- **VQ1.** What's the right escalation policy when verifier finds
  a spec defect? Options: (a) patch inline (this workgraph's
  approach), (b) reject back to spec-author with the finding,
  (c) escalate to architect if the defect implies a topology
  change. This workgraph used (a) for both F1 and F2; both were
  small. A bigger defect might warrant (b) or (c).
- **VQ2.** How exhaustive does file:line spot-checking need to
  be before "spot-check" stops earning its keep? 4 of ~20 caught
  none in this workgraph; if it had caught one, would the verifier
  have escalated to "check all"?
- **VQ3.** Mock-idiom mismatch (V1) is one specific case of
  "spec convention diverges from repo convention." Are there other
  conventions worth verifier-checking explicitly? Examples:
  test file location, error-class hierarchy, logging conventions,
  type-hint style. Probably worth a methodology pass through the
  repo's STYLE / CONVENTIONS docs to enumerate.

## References

- Spec files verified: `tasks/00-add-sdk-methods.md`,
  `tasks/01-impl-sigterm-graceful-release.md`,
  `tasks/02-impl-startup-reconciliation.md`,
  `tasks/03-integration-test-rolling-restart.md`.
- `vafi/tests/test_controller.py:15-32` — the existing
  `MockWorkSource` that revealed F1.
- architect-retro.md X3 (drift findings) — V3 spot-checks are
  drift-finding inputs.
- spec-author-retro.md SA4 (mock-spec discipline) — V1 is the
  verifier-side complement.
- spec-author-retro.md SAX3 (spec-author is the last gate before
  code) — VX2 refines this: verifier is *actually* the last gate;
  spec-author is the one-before-last.
- `viloforge-platform/docs/implementation-roadmap-PLAN.md`
  §"Phase 1" step 4 (Verify).
