---
stage: spec-author
workgraph: vafi-rolling-restart-fix
phase: methodology-extraction
authored_by: operator:lonvilo
co_authors:
  - claude-opus-4-7
created_at: 2026-05-14T00:00:00Z
session_duration_minutes: ~120
session_kind: interactive
tasks_specced:
  - t_cHx0UU (T0 — SDK methods)        # added DURING spec-author phase
  - t_4ZZ60T (T1 — SIGTERM unclaim)
  - t_QrwliD (T2 — startup reconciliation)
  - t_UlglCJ (T3 — integration test)
---

# Spec-author-phase retro — vafi-rolling-restart-fix

## Status of this document

Methodology-extraction scaffolding per `architect-retro.md` §X5
(corrected framing). Co-located in the workgraph dir; archived
or referenced from `vtf-methodologies/spec-author/*.md` once
that file stabilizes. NOT a standing SDD artifact.

## Purpose

Capture what worked, what surprised us, and what pattern-grade
findings emerged from the spec-author phase of this workgraph.
The downstream consumer is `vtf-methodologies/spec-author/bugfix.md`
(first draft) — refined once 2–3 more workgraphs accumulate
spec-author retros.

Phase 1 step 3 (spec-author) produced more methodology signal
than I expected. Of the 13 architect-level patterns (P1–P13)
documented in `architect-retro.md`, three (P11, P12, P13) were
ADDED post-session as direct consequences of spec-author-phase
findings. That's the rate this stage is generating gaps in the
architect-phase methodology. Real methodology data.

## Session timeline — raw substrate

1. **Spec-author T1 (SIGTERM unclaim) — opened with a
   conversational design point about "remaining real decisions."**
   I listed 9 decisions across T1/T2/T3 and labeled them
   spec-author scope.
2. **Operator pushback on the architect/spec-author boundary.**
   T3.1 (test environment: vafi-dev vs kind/k3d) was named as
   spec-author; operator flagged it as architect (evidential
   character). Resolved by promoting T3.1 to architect, locking
   ephemeral kind/k3d, adding **W1** + **P11** to retro. T3 task
   placeholder updated.
3. **T1 spec written** — `_current_claim_id: str | None`,
   finally-block unclaim before set_agent_offline, 7 acceptance
   criteria, full implementation sketch, MockWorkSource extension
   pattern, single unit test. Submitted as complete.
4. **Pre-T2 verification surfaced the SDK gap.** While grounding
   T2's `list_active_claims` call site, found that
   `AsyncTaskManager.unclaim` doesn't exist (only sync does);
   `AgentManager.tasks` doesn't exist on either side. This
   invalidated T1's "verified to exist" citation and forced
   adding **T0** to the DAG.
5. **W2 walk-back executed:** workgraph topology change (T0
   added; T1/T2 depend on T0), `target_repos` expanded,
   "Phase 1 single-repo" framing rewritten to "single-server-
   side-repo," **P12** added to retro, P3 amended, T1 spec
   patched.
6. **T0 spec written** — three SDK methods (`AsyncTaskManager.unclaim`,
   `AgentManager.tasks`, `AsyncAgentManager.tasks`) + tests +
   pin-bump coordination, 7 acceptance criteria.
7. **T2 spec written** — `_reconcile_orphaned_claims` method,
   `list_active_claims` protocol addition, 9 acceptance criteria
   including three unit tests (happy path, per-task error
   isolation, list-call failure isolation), multi-replica
   safety constraint encoded in docstring.
8. **Pre-T3 verification surfaced the CI gap.** While grounding
   T3, found vafi has no `.github/workflows/`; existing
   integration tests use `@pytest.mark.integration` + manual run
   against vafi-dev. Architect's locked "in CI on every PR" had
   embedded two claims; only the test-runtime claim was
   verified.
9. **W4 walk-back executed:** option (d) hybrid — Makefile target
   `integration-test-rolling-restart` spins up local kind, runs
   environment-agnostic test, tears down. CI wiring deferred to
   a follow-up `kind: infrastructure` workgraph. **P13** added
   to retro, **Q6** added to follow-up questions. Workgraph.md
   AC-4 + plan.md + task 03 updated.
10. **T3 spec written** — 9 acceptance criteria covering
    test + fixture + Make target + harness test-mode env var
    + cleanup + CI-out-of-scope.

## Spec-author-specific methodology candidates

Patterns to roll into `vtf-methodologies/spec-author/bugfix.md`
after 2–3 more workgraphs corroborate. Same maturity tagging
as architect-retro.

### SA1. Verify the full architect-asserted call chain at spec-author entry

**Maturity:** HIGH

**Observed.** Three out of four spec-author tasks (T1, T2, T3)
surfaced an architect-phase verification gap during their
prep. Each gap was on a *chain* the architect had touched only
at one end:

- T1/T2: SDK method existence (architect checked endpoint; SDK
  method didn't exist) → W2 → P12 in architect-retro.
- T3: CI delivery mechanism (architect checked test runtime; CI
  to run it didn't exist) → W4 → P13 in architect-retro.

The pattern: every architect call-chain claim is suspect until
verified end-to-end by spec-author.

**Candidate rule.** Spec-author's FIRST action for each task is
to enumerate every external-to-the-task-code call the spec
implies and verify each call's target exists in current code:

- Each API endpoint cited → grep the server-side route file.
- Each SDK method named → grep the SDK manager file (right
  sync/async variant for the agent's actual call pattern).
- Each test harness invoked → check it exists and runs in the
  expected way.
- Each CI/automation system referenced → check the workflow file
  exists, has the expected triggers, has access to needed
  secrets.
- Each file path mentioned → confirm the file exists in the
  target repo at the cited line range (`controller.py:81` etc.).

If any check fails, surface to operator before writing the
spec body. The walk-back is much cheaper at spec-author entry
than during executor implementation.

**Why this is a spec-author pattern, not architect.** Architect-
retro P12+P13 say "architect should verify the full chain." But
in practice the architect has dozens of decisions to make and
will sometimes miss the verification. Spec-author is the LAST
gate before executor; **spec-author owns the safety net.** The
architect's job is to *design* the right chain; spec-author's
job is to *verify* the designed chain still works.

### SA2. Spec frontmatter `acceptance_criteria:` is the contract; spec body is the rationale

**Maturity:** HIGH

**Observed.** Every task spec in this workgraph put SHALL/MUST
acceptance criteria in frontmatter (machine-readable) and the
implementation sketches/rationale in the Spec body (human-
readable). The frontmatter is what the judge grades against
per-criterion; the body is what the executor reads to know
*how* to satisfy each criterion.

**Candidate rule.** When writing a task spec:

1. Each acceptance criterion in frontmatter is a single SHALL/
   MUST statement that is independently verifiable by an
   external observer (judge or test). Avoid "X and also Y"
   — split into two criteria.
2. Implementation sketches go in the Spec body. They are
   NORMATIVE for the executor (representing the architect+
   spec-author's intended approach) but NOT the contract — the
   judge grades against acceptance criteria, not against
   whether the executor matched the sketch verbatim.
3. The split lets a creative executor find a better approach
   than the sketch as long as all ACs pass; it also lets a
   judge fail an implementation that matches the sketch but
   misses an AC.

**Edge case.** Sometimes a sketch encodes a constraint not
captured in any AC (e.g., a specific function name to avoid
naming collisions with an unrelated module). When that
happens, promote the constraint to a frontmatter AC or the
"Constraints" section — don't leave it implicit in the sketch.

### SA3. Cite real file paths with line numbers; never invent

**Maturity:** HIGH

**Observed.** Every file-path reference in the four task specs
was grounded against actual code at a specific line (e.g.,
`controller.py:81-95`, `vtf.py:197`, `views.py:269`). When the
architect's high-level reference was vague ("the controller's
register area"), spec-author resolved it to a specific line
range via direct file read.

**Candidate rule.** Specs name file paths AT THE LINE RANGE
they refer to. `controller.py` is wrong; `controller.py:81`
is right; `controller.py:81-95` is best when the spec refers
to a contiguous block.

**Why.** Specs are read by executors who haven't been in the
spec-author's head. A bare `controller.py` reference forces
them to grep; a line-range reference puts them in the right
spot immediately. Drift between spec and code (the file
changes) surfaces faster — a line range that no longer makes
sense is a visible defect, while a bare path silently rots.

**For methodology synthesis.** This is a low-cost discipline
with high downstream payoff. Strong candidate for the
methodology file's "minimum-quality-bar" section.

### SA4. Mock-double extension is part of the spec, not the executor's improvisation

**Maturity:** MEDIUM — first observation in T1/T2; corroboration
needed across more workgraphs to confirm it generalizes.

**Observed.** Each task spec that introduced a new `WorkSource`
protocol method also explicitly specified the corresponding
extension to the `MockWorkSource` test double in
`tests/test_controller.py`. The spec named:

- The new mock method body (e.g., "append to `unclaim_calls`,
  raise if `unclaim_errors[task_id]` is set").
- The new mock state attributes (e.g.,
  `unclaim_calls: list[str]`, `unclaim_errors: dict[str, Exception]`).
- Where to initialize them in the mock's `__init__`.

**Candidate rule.** If a task adds a protocol/interface method,
the spec MUST also specify the test-double extension. Leaving
mock extension to "the executor figures out the right pattern"
produces inconsistent mocks across tests and surprises the judge
when test patterns drift.

**Why.** The mock surface IS part of the public contract of the
protocol (every test needs to mock it the same way). Skipping
the mock spec leaks design decisions into "whoever writes the
first test."

**Edge case.** For protocols with many implementations across
the codebase, mock specification can become tedious. Could
defer to "follow existing mock conventions in
test_controller.py" if the conventions are well-established.
Worth observing whether this pattern survives multiple workgraphs
or whether shorthand emerges.

### SA5. "Out of scope" is normative, not decorative

**Maturity:** HIGH

**Observed.** Each task spec carries an "Out of scope" section.
This was used in this workgraph to:

- Explicitly defer cross-cutting concerns (multi-replica safety,
  pagination, periodic reconciliation).
- Document architect-level rejections (no `/release/` endpoint,
  no `tasks.reset` admin path).
- Capture rejected-but-tempting paths (no SIGINT-specific
  handling, no harness-resume work).

**Candidate rule.** "Out of scope" entries are normative — they
forbid the executor from doing the listed things even if they
seem like obvious next steps. The judge MUST flag a submission
that adds out-of-scope behavior as a violation, not as helpful
extras. Out-of-scope items often map to FUTURE workgraphs that
will be tracked separately.

**Why.** Without this discipline, executors with initiative
turn one workgraph into N+1 by "while I'm in here, I might as
well." That moves complexity into the wrong place and obscures
methodology signal.

### SA6. Frontmatter `depends_on:` is load-bearing infrastructure

**Maturity:** HIGH

**Observed.** When T0 was added to the DAG mid-spec-author,
the `depends_on:` field in T1, T2, T3 became the mechanism
that wired the topology together. The DAG's correctness is
*entirely* in those frontmatter fields.

**Candidate rule.** `depends_on:` is set at architect time
based on the topology mermaid. Spec-author can ADD entries
when discovering new dependencies (e.g., T1 needed T0 once T0
was added), but MUST NOT REMOVE entries without flagging to
operator — removal changes the DAG's executable shape.

**Implication for verifier phase.** A holistic check is that
every `depends_on:` ID resolves to a real task file in the
same workgraph dir, and the resulting graph is acyclic. The
verifier phase mechanically validates this.

## Cross-cutting findings (any ingestion stage)

### SAX1. The walk-back IS the methodology data

**Observed.** Three walk-backs (W1, W2, W4) in one workgraph,
all P12/P13-shaped (architect verified one end of a chain but
not the full chain). The walk-backs aren't noise to be
suppressed; they're the highest-signal output of Phase 1.

**Cross-cutting form.** Every ingestion stage during a
methodology-extraction phase should TAG walk-backs explicitly
in its retro and link them to the methodology patterns they
produced. Tag them with a date so chronological refinement is
traceable.

When `vtf-methodologies/<role>/<kind>.md` is eventually
synthesized from N workgraphs' worth of retros, the walk-back
density per phase is a primary signal of where the methodology
needs to put its weight.

### SAX2. Investigation tooling needs to support full-chain tracing

**Observed.** Architect-retro Q3 asks "what tooling does the
autonomous architect agent's investigation pass need?" The W2 +
W4 walk-backs answer with specifics:

- **SDK-surface introspection** — given a server endpoint, list
  the SDK methods that wrap it (or report none).
- **Test-delivery tracer** — given a "this test runs in CI"
  claim, verify the CI workflow file exists and triggers on
  the expected events.
- **File-path validator** — given a path:line citation, verify
  it points at code with the expected shape.
- **Mock surface diff** — given a protocol change, predict
  which test doubles need extension.

Without this tooling, the autonomous architect agent will
reproduce the W2/W4 mistakes at scale. With it, it will catch
them at investigation time.

### SAX3. Spec-author is the methodology's last gate before code

**Observed.** Each architect-phase verification gap (W2, W4)
surfaced during spec-author prep, NOT during executor
implementation. If it had surfaced during executor phase, the
cost would have been: executor blocked, workgraph backed out
to architect, redo, judge re-grades, full loop. Surfacing
during spec-author was: pause for 20 minutes, update artifacts,
proceed.

**Cross-cutting form.** Spec-author isn't a transcription
stage — it's a verification stage. The methodology should
treat spec-author's responsibility as: *prove the architect's
designed chain still works against current reality, OR flag
walk-backs.* That responsibility belongs nowhere else.

**Implication.** Spec-author retros are at least as
methodologically valuable as architect retros — and should be
synthesized into the autonomous spec-author agent's prompt
with equal weight to the architect's. SA1 (full-chain
verification at task entry) is the load-bearing rule.

## Open questions for the next workgraph

- **SAQ1.** Does SA1 (full-chain verification at task entry)
  remain HIGH-maturity after more workgraphs? Specifically:
  do the next workgraphs find architect-phase verification
  gaps at the same rate (3 of 4 tasks) or lower? If lower, the
  rule is doing its job by tightening the architect-phase
  methodology too. If higher, SA1 IS the spec-author
  methodology.
- **SAQ2.** When the executor finds a real implementation
  obstacle (e.g., a library version conflict), does that
  surface as another walk-back at executor phase? Or does
  spec-author's chain-verification (SA1) catch it
  proactively? Worth observing — the answer shapes whether
  executor-retros are also needed during methodology-
  extraction phases.
- **SAQ3.** SA4 (mock-double extension is part of spec) was
  HIGH-maturity in this workgraph but could become tedious
  at scale. Worth observing whether shorthand emerges
  ("follow existing mock conventions in `<file>`") and
  whether the shorthand erodes mock consistency.

## References

- workgraph.md (this directory) — final shape: kind: bugfix,
  flat 4-task DAG (T0 → T1, T0 → T2, {T1,T2} → T3), 2
  target_repos (vtaskforge for T0; viloforge/vafi for T1-3).
- plan.md (this directory) — includes cross-repo walk-back
  section and the operator decisions resolved during
  spec-author.
- architect-retro.md — W1/W2/W4 walk-backs that motivated
  P11/P12/P13. SA1 here is the spec-author-side complement to
  the architect-side P12/P13.
- Task specs: `tasks/00-add-sdk-methods.md`,
  `tasks/01-impl-sigterm-graceful-release.md`,
  `tasks/02-impl-startup-reconciliation.md`,
  `tasks/03-integration-test-rolling-restart.md`.
- `viloforge-platform/docs/project-repo-DESIGN.md` §"tasks/<NN>-<slug>.md"
  — the spec template structure these task specs follow.
- `viloforge-platform/docs/implementation-roadmap-PLAN.md`
  §"Phase 1 — Manual SDD validation" step 3 (spec-author).
- User memories applied this stage:
  `feedback_verify_before_claiming`,
  `feedback_discuss_points_conversationally`,
  `feedback_proceed_per_protocol`,
  `feedback_no_ai_attribution_in_commits`,
  `feedback_autonomous_on_reversible`.
