---
stage: architect
workgraph: vafi-rolling-restart-fix
authored_by: operator:lonvilo
co_authors:
  - claude-opus-4-7
created_at: 2026-05-14T00:00:00Z
session_duration_minutes: ~90
session_kind: interactive
---

# Architect-phase retro — vafi-rolling-restart-fix

## Purpose

This is the **architect-phase retro** for the first workgraph
authored under SDD discipline. It is NOT the workgraph-completion
retro — that comes after execution+judge and lives in
`completion.md`. This document captures what we learned about
**driving the architect phase well** so the eventual autonomous
architect agent and future operator-driven sessions inherit the
heuristics.

Three audiences:

1. **The autonomous architect agent** — its system-prompt /
   methodology file in `vtf-methodologies/architect/bugfix.md`
   (eventually) needs to encode the heuristics that worked here.
   This document is first-draft input to that file.
2. **Future operators driving interactive architect sessions** —
   a playbook so subsequent sessions don't re-derive what worked.
3. **Other ingestion-agent designers** (triage, spec-author,
   verifier, judge) — several findings here cross-cut, and the
   cross-cutting section is for them.

## Session timeline — raw substrate

The factual record of what happened. Future syntheses should be
able to back-trace any extracted pattern to a concrete moment here.

1. **Cold-start orientation.** Operator typed "lets contine [sic]
   the work on viloforge". Per `feedback_kb_recent_first_on_resume`,
   ran `kb recent` → identified the viloforge-platform workspace's
   active handoff (Thread C — vafi#4 architect session). Loaded
   workspace via `kb work start`.

2. **Handoff orientation gap.** Handoff pointed at orientation docs
   in `viloforge-projects-vafi/docs/INDEX.md` etc.; that directory
   was empty. The docs actually live in
   `viloforge-platform/docs/`. Resolved via `find` rather than
   asking — minor friction, surfaced as observation O1 below.

3. **Pre-existing doc contradiction discovered.** Found that
   `implementation-roadmap-PLAN.md:176` explicitly says
   `kind: bugfix` for vafi-rolling-restart-fix while
   `project-repo-DESIGN.md:898` shows the same workgraph as
   `kind: feature` in the worked example. Surfaced to operator
   as design point #1. Operator chose: fix the upstream doc.
   Both refs updated, conflict resolved.

4. **Topology straw-man presented.** Three viable shapes
   (Option A flat 3-task, Option B canonical split 5-task, Option C
   shared-design 4-task). Recommended Option A with reasoning
   (architectural design already exists in `vafi-runtime-DESIGN.md`;
   spec-author phase produces the per-task design layer the worked
   example shows in a separate task; bugfix tier should resist
   feature-tier ceremony). Counter-argument considered and
   articulated (worked-example pattern might be load-bearing for a
   downstream judge artifact).

5. **Endpoint sub-question flagged.** Noticed `vafi-runtime-DESIGN.md`
   named aspirational endpoints (`/release`, `agents/<id>/claims`)
   that we hadn't verified existed. Coupled this to design point #3.

6. **Operator triggered `ultrathink` at the locking moment.** Caused
   a deeper investigation pass: searched vtaskforge views, agents
   urls, SDK managers. Discovered:
   - `POST /v2/tasks/<id>/unclaim/` exists
     (`vtaskforge/src/tasks/views.py:269`) with exactly the right
     semantics: `doing → todo`, clears claim fields, emits
     dedicated `unclaimed` event, state-machine-guarded.
   - `POST /v2/tasks/<id>/reset/` is documented as
     "**Admin force-transition: bypass state machine**" — wrong
     tool for normal agent claim-release.
   - SDK exposes `client.tasks.unclaim`, `client.tasks.reset`,
     and `client.agents.tasks` — all needed methods already
     present.
   - Agent endpoint filters by `claimed_by=agent.user` (user-level
     not agent-instance-level) → multi-replica safety constraint.

7. **Design-point revisions.**
   - #3 endpoint choice: revised from "reuse `reset`" → "use
     existing `unclaim`". Better tool than the handoff or the
     runtime DESIGN had identified.
   - Upstream doc fix: `vafi-runtime-DESIGN.md` rewritten to
     reflect as-built (commit `ef48c8f`).

8. **Artifact authoring.** Per the file-template conventions in
   `project-repo-DESIGN.md`: wrote `obs_jsWWsQ.md` (triage
   verdict promoted_new), `workgraph.md` (kind:bugfix, flat
   topology, 6 SHALL/MUST DAG-level criteria), `plan.md`
   (approach, 5 considered alternatives, risks, open-questions
   resolved), three task placeholders (frontmatter only,
   body TBD).

9. **Commit + push.** Two commits — `ef48c8f` (viloforge-platform
   doc edits) and `33cbcea` (viloforge-projects/vafi artifacts).

### Notable session-specific observations

- **O1.** Handoff pointed at orientation paths that didn't exist
  in the target repo; orientation docs lived in the platform repo.
- **O2.** `kind: feature` vs `bugfix` contradiction had sat in
  the docs pre-session; surfaced and fixed.
- **O3.** Runtime DESIGN named two aspirational endpoints; neither
  existed.
- **O4.** Investigation found `unclaim` was the right call — neither
  handoff nor runtime DESIGN had identified it.
- **O5.** `agents/<id>/tasks/` filters by user, not by agent instance;
  multi-replica constraint surfaced and documented.
- **O6.** Controller already had a signal handler at
  `controller.py:532` that only sets `_shutdown` — the fix is an
  extension of an existing hook, not new installation.
- **O7.** SDK had all needed methods already wired
  (`tasks.unclaim`, `agents.tasks`).
- **O8.** Phase 1 single-repo constraint became materially testable
  via the `unclaim` discovery; without it we would have had a
  cross-repo question to resolve.
- **O9.** Triage was done by operator (not by a triage agent);
  noted in `obs_jsWWsQ.md` frontmatter.
- **O10.** Two commits pushed.

## Walk-backs (post-session corrections)

Findings surfaced after the architect session formally closed that
required updating this retro and/or the workgraph artifacts. Captured
explicitly so the methodology-extraction step can see *which patterns
emerged from real walk-backs* vs *patterns that were obvious during
the session*.

- **W1 — Test environment was architect-level, not spec-author** (date:
  2026-05-14, post-session). Surfaced when operator asked
  "which of these decisions are spec-author?" and I had listed T3.1
  (vafi-dev cluster vs kind/k3d) as spec-author. The decision changes
  what passing the test *proves*, which is the architect's evidential-
  character concern, not spec-author's how-to-build-the-test concern.
  Resolved: locked to ephemeral kind/k3d in CI; workgraph.md AC-4 +
  plan.md + task 03 updated. Methodology finding: **P11**.

- **W2 — Async SDK surface gap missed during architect investigation**
  (date: 2026-05-14, post-session, during spec-author T2 prep). The
  architect-phase "investigation pass" (P6) verified that
  `unclaim` and `agents/<id>/tasks/` *endpoints* exist in vtaskforge
  but did NOT verify that the async SDK exposes them. Reality:
  `AsyncTaskManager.unclaim` and both sync/async `AgentManager.tasks`
  do not exist; only sync `TaskManager.unclaim` exists. This
  invalidated the workgraph's "Phase 1 single-repo" framing and
  forced adding **T0** (`00-add-sdk-methods.md`) — a vtaskforge-repo
  task — to the DAG. Methodology finding: **P12**. Walk-back impact:
  workgraph.md `target_repos` expanded to `[vtaskforge, viloforge/vafi]`;
  plan.md gained a cross-repo walk-back section; AC-5 reframed from
  "no vtaskforge changes" to "no vtaskforge **server-side** changes."

- **W3 — Stage-retros are extraction scaffolding, not standing SDD**
  (date: 2026-05-14, post-session). Operator-flagged correction
  to X5. The retro originally framed per-stage retros as a permanent
  SDD pipeline artifact alongside `completion.md`; the corrected
  framing is that they exist only during methodology-extraction
  phases. See X5 body below.

The walk-backs *are* the methodology data Phase 1 was designed to
produce. The architect session closing with W1-W3 still latent means
the architect-phase methodology has gaps the autonomous agent will
need to close (P11, P12 specifically address those gaps; W3 was a
framing error of mine that the operator corrected).

## Architect-specific methodology candidates

Provisional patterns to roll into `vtf-methodologies/architect/bugfix.md`
after 2–3 more workgraphs have produced corroborating retros. Each
pattern has a maturity tag: **HIGH** (strongly supported by this
session), **MEDIUM** (supported but worth corroborating), **LOW**
(speculative, needs more data).

### P1. Cold-start orientation must include cross-doc consistency check

**Maturity:** HIGH

**Observed.** The session opened with an unflagged contradiction
(`kind: feature` vs `bugfix`) between two docs the architect must
read. The handoff also pointed at paths that didn't exist where
named. Both were caught by reading critically rather than by reading
to absorb.

**Candidate rule.** Before locking any design decision, the architect
enumerates the docs it must trust (handoff, project DESIGN, runtime
DESIGN, implementation roadmap, the issue body) and explicitly
checks them against each other for naming/version/scope contradictions.
Contradictions are surfaced to the operator as the FIRST design
point of the session, before topology questions.

**Edge cases / when not to apply.** None obvious. Even trivial
workgraphs benefit from a 60-second consistency check; the cost is
low and the value is high (a contradiction discovered mid-session
is much more expensive to resolve than one discovered up front).

**How to validate further.** In subsequent workgraphs, log every
contradiction the architect finds at orientation time. If most
sessions find zero, this becomes lower-priority; if most sessions
find one or more, it's load-bearing methodology.

### P2. For bugfix-tier with pre-existing architectural design, prefer flat DAG

**Maturity:** MEDIUM — strong in this session but generalizes only
across "bugfix tier with existing design doc" cases, which need
more data points.

**Observed.** vafi-rolling-restart-fix's architectural design was
already complete in `vafi-runtime-DESIGN.md`. The worked-example
pattern in `project-repo-DESIGN.md` shows split design+impl tasks
(`01-design-handler.md` + `02-impl-sigterm.md`). We chose flat
because the design layer the split pattern would produce was
duplicative of (a) the upstream DESIGN doc and (b) the spec-author
phase's per-task acceptance-criteria work.

**Candidate rule.** When the architectural design already exists in
an established design doc (any `*-DESIGN.md` referenced from
`project.yaml`), the architect prefers a **flat DAG**: impl tasks
plus a verification task. The split design+impl pattern is
appropriate when:
- The design itself is novel and non-trivial (feature tier);
- The bugfix's design is ambiguous or contested in the existing
  design doc;
- Multiple impl tasks need a shared design artifact that
  doesn't yet exist anywhere.

**Edge cases.** Even at bugfix tier, if the architect finds that
the existing design doc is wrong or incomplete *for the specific
fix at hand*, a design-resolving task at the head of the DAG is
warranted — but its output should be a patch to the upstream
design doc, not a separate forever-living spec.

**How to validate further.** Next two workgraphs (one feature,
one refactor) should consciously NOT apply this rule and check
whether the split pattern adds value there. If the feature
benefits from split and the bugfix didn't, the rule holds.

### P3. Endpoint/API choices must be verified against code, not against doc-level names

**Maturity:** HIGH

> **Amended after W2 (see Walk-backs).** P3 as originally written
> was too narrow — it caught doc-vs-server-endpoint drift but not
> server-vs-SDK drift. The full version is in P12. P3 stays as the
> umbrella discipline ("verify against code"); P12 is the
> concrete refinement covering the full call chain.

**Observed.** The handoff said use `tasks.reset(...)`. The runtime
DESIGN said use `POST /v2/tasks/<id>/release/`. Neither was right —
`unclaim` was the correct endpoint, and only checking
`vtaskforge/src/tasks/views.py` surfaced it. Aspirational naming in
design docs (where authors propose what a clean endpoint should
look like) is a different surface from the as-built API.

**Candidate rule.** Before naming any external API endpoint, SDK
method, file path, or function in `workgraph.md`/`plan.md`/task
specs, the architect verifies the name against the actual
implementation in the linked target repo (or vendored dependency
repo). This is a grep, not a "I think this is right" step.

**Why this matters more than it sounds.** Doc-level names get
propagated through workgraph.md → task specs → executor work.
If the architect locks the wrong name, every downstream artifact
inherits the error and the spec-author/judge cycle is needed to
catch it — much later, much more expensively.

**Edge cases.** If the design genuinely requires adding a new
endpoint, the architect should propose that explicitly (with
which repo, what scope), but the workgraph then becomes
multi-repo, which often violates declared scope constraints
(see P7).

### P4. Conversational design points, not bundled questions

**Maturity:** HIGH — also matches user-memory `feedback_discuss_points_conversationally`

**Observed.** Each design point in this session was presented as
plain text: "Here's the conflict. My read is X. Reasoning: ...
Counter-arg I considered: ... Your call?" Operator could redirect
each one independently. When `AskUserQuestion`-style bundling
would have been tempting (kind + topology + endpoint together in
one form), staying conversational let us surface the
endpoint-investigation finding mid-flow.

**Candidate rule.** The interactive architect protocol with
operator is one-design-point-at-a-time, in plain text:

1. Restate the question.
2. Recommend with reason.
3. Articulate the counter-argument considered.
4. Ask explicitly: "Your call?"
5. Wait. Don't proceed to the next point until this one's
   answered (or until operator says "lock all defaults, proceed").

**Edge cases.** Pure factual sub-questions ("what's the file
path of X?") can be batched with the design point they belong
to. The rule applies to **decisions**, not lookups.

**For the autonomous architect agent variant.** This translates
to: the agent emits one decision-point summary at a time to the
operator-loop API, with a structured `recommendation`,
`reason`, `counter`, and waits for an
`accept | redirect | clarify` response before proceeding.

### P5. Recommend with a reason and a counter-argument considered

**Maturity:** HIGH

**Observed.** Every design point in this session was answered by
operator within a few lines because the recommendation was
defensible. The counter-argument articulation prevented the
operator from having to invent their own pushback — they could
either accept the recommendation, accept the counter, or
introduce something neither covered.

**Candidate rule.** Never present a design point as a neutral
"what do you want?" The architect always:

- States its recommendation.
- Names the strongest reason for it.
- Names the strongest counter-argument it considered.
- Discloses what would change its mind (the conditions under
  which it would switch).

**Why.** This shifts operator effort from "synthesize a
recommendation" to "ratify or redirect" — a much cheaper
operation, and the one operators are actually good at.

### P6. Deeper-investigation pass before locking irreversible structural choices

**Maturity:** HIGH — directly produced the biggest correction in
this session.

**Observed.** Operator said `ultrathink` at the topology-locking
moment. That triggered me to actually check vtaskforge views/SDK
for the proposed endpoints. The result: `unclaim` discovery,
`reset` rejected as an admin escape hatch, runtime DESIGN
corrected, Phase 1 single-repo constraint preserved.

**Candidate rule.** Before writing `workgraph.md` frontmatter, the
architect enters an explicit "investigation pass" that re-verifies:

- Endpoint names exist in target repo code (P3).
- File paths and function signatures referenced by tasks exist.
- Constraint citations match the source docs.
- Scope-constraint compatibility (P7) is actually achievable.
- No upstream doc contradictions remain unresolved (P1).

If the investigation pass surfaces a finding that changes the
design, the architect surfaces it to the operator before locking,
not after.

**For the autonomous architect agent variant.** This is a
mandatory pre-locking step, not operator-triggered. Equivalent to
a built-in "extended thinking" budget allocated before the
final architect output is committed.

### P7. Treat declared scope constraints as filters on the solution space

**Maturity:** HIGH

**Observed.** Phase 1's "single-repo" constraint was at risk when
the obvious paths (reuse `reset`; add `/release`) either were
wrong or required vtaskforge changes. Treating the constraint as a
filter — "what solution exists inside this constraint?" — led
to the `unclaim` discovery.

**Candidate rule.** When a scope constraint is declared in a
roadmap or `project.yaml`, the architect actively searches for
solutions inside the constraint **before** proposing solutions
that violate it. If no in-constraint solution exists after a
thorough search, the architect surfaces both the in-constraint
search results AND the proposed violation, letting the operator
choose explicitly.

**Anti-pattern.** Silently widening scope ("oh we'll just add
this to the other repo") without explicit operator ratification.

### P8. When you find an upstream doc contradiction, fix the doc, not just work around it

**Maturity:** HIGH

**Observed.** The `kind: feature` vs `bugfix` conflict could have
been worked around silently — pick one, move on. We fixed the
upstream DESIGN doc instead. Result: the contradiction won't
re-surface for the next architect session.

**Candidate rule.** Documentation drift is a self-replicating
defect. When the architect finds a contradiction between two
upstream docs (or between a doc and as-built code), it proposes
a fix to the more-stale or more-incorrect side as part of the
workgraph's commit set. The fix is bundled with the architect
session's output, not deferred.

**Edge cases.** If the fix is contentious (e.g., requires a
design decision the operator hasn't made), the architect
surfaces the contradiction explicitly and lets the operator
decide which side to align. Either way, the contradiction
doesn't survive the session unflagged.

### P9. Tag methodology-relevant findings explicitly during planning

**Maturity:** MEDIUM

**Observed.** `plan.md` includes an "Open questions resolved
during planning" subsection that captures the decision rationale
for kind, topology, endpoint, and design-split — making them
traceable from retro / completion.md later.

**Candidate rule.** `plan.md` always includes:

- "Open questions resolved during planning" — decisions made,
  with rationale.
- "Risks" — known unknowns the architect flagged but did not
  resolve.
- "Considered alternatives" — competing options and why
  rejected.

This is already in the file template; the rule here is that
the architect must POPULATE these sections, not leave them
empty.

**For methodology synthesis.** These three subsections become
the primary input for `vtf-methodologies/architect/<kind>.md`
synthesis across N workgraphs.

### P12. Verify the FULL call chain from agent code to server, not just the server endpoint

**Maturity:** HIGH — directly caused the W2 walk-back; methodology
gap was unambiguous.

**Observed.** Architect-phase investigation under P6 ("Deeper-
investigation pass") and P3 ("Endpoint/API choices must be verified
against code") verified that `POST /v2/tasks/<id>/unclaim/` and
`GET /v2/agents/<id>/tasks/` existed in vtaskforge's server-side
views. Both findings were correctly logged in workgraph.md and
plan.md. **What was NOT verified: whether the SDK the agent
actually calls (`vtf-sdk-python`, async client) exposed these
endpoints.** Reality: `AsyncTaskManager.unclaim` does not exist
(only the sync `TaskManager.unclaim`); neither sync nor async
`AgentManager` has a `tasks(...)` method. The agent's actual call
chain has a gap at the SDK layer that the architect didn't see.

The gap surfaced during spec-author T2 prep when I needed to
specify the exact SDK call vafi would make and discovered it
didn't exist. That triggered the walk-back: workgraph topology
gained T0 (a vtaskforge-targeted task), `target_repos` expanded,
the "Phase 1 single-repo" framing was rewritten as "single-server-
side-repo plus SDK-companion alignment," and AC-5 was reworded.

**Candidate rule.** Before locking the workgraph topology or
declaring scope constraints satisfied, the architect verifies the
**entire call chain** the agent will traverse:

1. Server-side endpoint exists at the URL and HTTP verb claimed.
2. The SDK / client library / wire wrapper the agent uses
   *exposes* that endpoint via a method the agent can actually
   call.
3. For each repo-boundary that wrapper crosses (pip pin, vendored
   submodule, gRPC stub, generated client), confirm the version
   the agent is pinned to includes the method.

If step 2 or step 3 fails, the architect either:

- Adds an explicit task to the DAG that closes the gap (`T0` here),
  with appropriate `target_repo` and `depends_on` wiring; OR
- Reframes the workgraph's scope (e.g., target a different
  endpoint that's fully available); OR
- Declares the gap as an out-of-scope precondition the operator
  must close before the workgraph can be `ready`.

What the architect MUST NOT do: silently assume the SDK will
expose the endpoint because the endpoint exists. That assumption
pushes discovery into spec-author phase (where this finding
surfaced) or executor phase (worse — the executor would have hit
an AttributeError mid-implementation).

**Edge cases.**

- **Pure HTTP-direct callers** (no SDK abstraction) skip steps 2-3.
- **Multi-language SDKs** (e.g., a service called from both Python
  and Go) — verify the language the executor will use, not just
  the most-popular one.
- **Sync/async splits** (the specific failure mode here) — verify
  the variant the agent uses, not just the more-prominent one.

**How to validate further.** In subsequent workgraphs:

1. Log every "SDK surface verified" finding the architect makes at
   investigation time, separately from "endpoint exists" findings.
   If most workgraphs find them aligned, this rule is easy. If
   most workgraphs find SDK gaps the architect missed, the
   investigation tooling needs to be stronger (P6 needs richer
   tooling — see Q3).

2. The autonomous architect agent's investigation tooling must
   include SDK-surface introspection, not just code grep. Suggested
   tool surface (rough sketch):
   - `sdk_method_exists(repo, manager_class, method_name) -> bool`
   - `sdk_call_path(endpoint_url) -> list[SDKMethod | "missing"]`

**Cross-link to P3.** P3 should be amended: "verify the FULL call
chain, not just the server endpoint." The original P3 wording
("verify endpoint names against actual implementation") is too
narrow — it admits the failure mode this pattern names. P3 stays
as the higher-level discipline; P12 is the concrete refinement.

### P11. Architect owns evidential-character decisions, not just topology

**Maturity:** HIGH — surfaced by post-session operator question on
the architect/spec-author phase boundary.

**Observed.** When listing remaining open decisions for spec-author,
T3.1 (test environment: vafi-dev real cluster vs ephemeral kind/k3d
in CI) was listed as a spec-author decision. Operator flagged that
this was mis-categorized. Reason: the test-environment choice
changes what we *conclude* from a passing test (controller-lifecycle
correctness vs deployment-shape-under-ArgoCD correctness) — that's
the *evidential character* of the validation, which is the
architect's call. Plan.md even had a giveaway line ("Spec-author
phase resolves: which test environment") that should not have
existed. Resolved post-session: locked to ephemeral kind/k3d in CI
(orphan-recovery is controller-lifecycle, not deployment-shape).
Workgraph.md AC-4 + plan.md T3 + task 03 placeholder all updated.

**Candidate rule.** For any task in the DAG that involves
validation (tests, audits, measurements, benchmarks), the architect
must decide the validation's *evidential character* — what kind of
evidence is acceptable, against what environment, with what
boundary conditions — before handing off to spec-author. Spec-author
implements the test; architect specifies what passing the test
*proves*.

**Distinguishing rule.** If changing the answer changes what we
*conclude* from passing the test, it's architect. If it only
changes how we *build* the test, it's spec-author.

Examples:

- "kind/k3d vs vafi-dev cluster" → architect (different evidence)
- "kind vs k3d" → spec-author (same evidence; pick whichever is
  standardized)
- "poll-and-timeout vs event-stream assertion" → spec-author
  (same evidence)
- "unit-test only vs integration test" → architect (different
  evidence)
- "test environment variables to inject" → spec-author
- "what passing the test allows us to ship to production" →
  architect

**Why the line matters.** Spec-author has neither the cross-doc
visibility nor the scope-constraint visibility the architect has.
Asking spec-author to make evidential-character calls means asking
the wrong stage to weigh against e.g. Phase 1 single-repo
constraints, future-workgraph dependencies, or roadmap timelines.

**For the autonomous architect agent variant.** The agent's
checklist before emitting task placeholders must include "for
each validation-related task, is the evidential character
specified?" An unspecified evidential character is a defect that
verifier should also catch.

### P10. Use real IDs in task placeholders even when bodies are TBD

**Maturity:** HIGH

**Observed.** Task IDs (`t_4ZZ60T`, `t_QrwliD`, `t_UlglCJ`) were
generated and locked into frontmatter even though the bodies
were placeholder "TBD — to be spec-authored". The
`depends_on` edges in T3 reference these IDs.

**Candidate rule.** The architect generates task IDs at the
time it lays out the DAG topology. IDs are stable across the
spec-author phase; only bodies and `acceptance_criteria`
change. Once an ID is in frontmatter, downstream `depends_on`
references rely on it.

**Implication for the autonomous architect agent.** ID
generation is part of the architect's deliverable, not the
spec-author's. The agent should never emit a task placeholder
without a stable ID.

## Cross-cutting findings for other ingestion-agent designers

Patterns that surfaced in this session but apply to triage,
spec-author, verifier, and judge as well. These should NOT live
buried inside `vtf-methodologies/architect/` — they belong in
a shared protocol layer (perhaps
`vtf-methodologies/_shared/verification-discipline.md` or
similar).

### X1. Verify-before-claim discipline applies to every ingestion stage

**Observed in this session.** P3 above (endpoint names verified
against code, not docs) is the architect's expression of this
discipline.

**Cross-cutting form.**

- **Triage** should verify that its classification is supported
  by actual workspace state — open similar issues, existing
  workgraphs in any lifecycle state — not by issue framing
  alone. (Maps to project-repo-DESIGN.md's existing triage
  process which already names this; reinforced here.)
- **Spec-author** should verify file paths exist in
  `target_repo`, function signatures match, and acceptance
  criteria reference real test infrastructure. Aspirational
  paths leak from design docs straight into specs otherwise.
- **Verifier** should walk holistic checks against the actual
  workgraph contents (DAG topology validity, methodology pin
  correctness, zero `[NEEDS CLARIFICATION]` markers), not
  against a checklist of what should be present.
- **Judge** grades against the actual submitted artifact's
  observable behavior (tests passing, code behavior matching
  acceptance criteria), not against the executor's
  self-assessment text.

**Maps to user memory.** `feedback_verify_before_claiming`,
`feedback_test_mcp_end_to_end` — the discipline is already
familiar at the workstation level; the ingestion-agent
methodology should encode it explicitly per role.

### X2. Triage should encode design-context pointers, not just classification

**Observed in this session.** `obs_jsWWsQ.md` included not just
the `promoted_new` outcome but also: "architectural design
already exists in vafi-runtime-DESIGN.md; bugfix-tier execution
of that design." That framing pre-shaped P2 (flat DAG) and
saved the architect phase real work.

**Cross-cutting form.** Triage's output isn't just an outcome
verdict — it should pre-tag the observation with:

- Relevant existing design docs (which sections).
- Relevant existing decisions and gotchas in the project's
  knowledge base.
- Any declared scope constraint that pre-shapes the workgraph
  (single-repo? specific phase?).
- Tentative `kind:` (the architect can override, but the
  pre-tag is a useful prior).

**Implication for the autonomous triage agent's prompt.** Its
methodology should include "search for similar prior workgraphs,
referenced design docs, and existing knowledge-base entries
relevant to this observation before writing the outcome."

### X3. Doc–reality drift is a recurring failure mode; every stage should surface it

**Observed in this session.** Two drifts caught in one session:
`kind: feature` vs `bugfix` (doc–doc); `release`/`reset` vs
`unclaim` (doc–code).

**Cross-cutting form.** Every ingestion+execution stage should
emit a structured "drift findings" output when it finds any:

- **Triage**: "this issue references X which doesn't match
  codebase state".
- **Architect**: "two design docs disagree on this entity's
  name/shape" / "design doc names an API that doesn't exist".
- **Spec-author**: "the file path in `workgraph.md` doesn't
  exist in `target_repo`".
- **Verifier**: "the methodology pinned doesn't match the
  workgraph kind".
- **Judge**: "the submitted artifact references a path the
  spec didn't name".

**Methodology implication.** Drift findings shouldn't be buried
in retros; they should be a first-class output the runtime
collects and surfaces. Over time, the drift-findings stream
across all workgraphs becomes a measure of doc-health.

### X4. Operator interrupts are valuable; the agent loop must signal where it's about to lock something

**Observed in this session.** The `ultrathink` interrupt at the
topology-locking moment produced the largest design improvement
of the session.

**Cross-cutting form.** Every ingestion stage in interactive
mode should explicitly signal to the operator before:

- Locking an irreversible structural choice (architect: DAG
  topology / task IDs; triage: outcome verdict).
- Committing an artifact that downstream stages will rely on
  (spec-author: per-task acceptance criteria).

The signal opens an interrupt window the operator can use to
redirect or request deeper investigation (P6). For the
autonomous agent variant, this is the natural place to insert
a "pause for review" gate in supervised modes.

### X5. Stage-retros are methodology-extraction scaffolding, not standing SDD artifacts

**Observed in this session.** This document is a stage-retro
(architect-phase only). It exists because we are in Phase 1 of
the implementation roadmap — explicitly tasked with extracting
first-draft methodology files into `vtf-methodologies`. The
canonical SDD retro artifact, per `project-repo-DESIGN.md`, is
workgraph-level `completion.md`. Stage-retros are not part of the
standing SDD pipeline.

**Corrected framing (operator-flagged 2026-05-14).** During
**methodology-extraction phases** (Phase 1; then later when a new
`kind:` variant — feature, refactor, infrastructure, research,
trivial — gets its first workgraph and needs methodology drafted),
we retain per-stage retros as scaffolding. They serve methodology
extraction. Once the methodology file for that (role × kind)
pair stabilizes, future workgraphs of that kind stop producing
per-stage retros — the durable learning lives in the methodology
file, not in retro accumulation.

**Where they live during Phase 1 (and equivalent later pockets).**
Inside the workgraph directory, alongside `workgraph.md` /
`plan.md` / `completion.md`:

- `architect-retro.md` — this document.
- `spec-author-retro.md` — written at end of spec-author stage
  during methodology-extraction phases only.
- `verifier-retro.md`, `judge-retro.md` — same pattern.
- `triage-retro.md` — only if triage produced learning worth
  capturing (one-line operator decisions skip).

Co-location (vs a separate scratch area) was chosen so each retro
sits next to the artifact it comments on; this makes methodology
extraction easier ("re-read the architect-retro alongside the
workgraph.md it was about"). Trade-off accepted: future readers
might assume retros are canonical SDD; the retro frontmatter
should make the experimentation status explicit (a `stage:` and
`phase: methodology-extraction` marker would help — worth
considering for the spec-author-retro template).

**Post-extraction life.** Once `vtf-methodologies/<role>/<kind>.md`
files exist and are good enough to drive the autonomous agents,
the stage-retros become **historical archaeology** — useful for
"why is the methodology this shape?" questions, but not consumed
by the standing pipeline. They can be left in place as a record
or archived as a batch; not deleted under sentimental value.

**Why this correction matters.** Framing per-stage retros as
"standing SDD" overreaches. SDD's contract with future workgraphs
is `completion.md` plus the methodology files. Stage-retros are
the dev-loop for methodology, and dev-loop artifacts shouldn't
masquerade as production schema.

## Open questions for the next workgraph

Things this session left unresolved that the next workgraph should
deliberately exercise:

- **Q1.** Does P2 (flat DAG for bugfix) hold for non-pre-designed
  bugfixes? Pick a bugfix where the design isn't already done
  upstream, and consciously try flat. Report.
- **Q2.** Does the split design+impl pattern produce real value
  at feature tier? Pick the `kind: feature` follow-up workgraph
  (per roadmap, "a small new ergonomic to vfdash") and
  consciously apply the split pattern. Compare ceremony cost vs
  value vs. flat.
- **Q3.** What does the autonomous architect agent's
  "investigation pass" (P6) need as tooling? Specifically: code
  search (grep), endpoint discovery (OpenAPI spec or
  view-introspection), and doc-cross-reference. Worth a small
  spike before the agent is built.
- **Q4.** How does the multi-replica safety constraint
  (out-of-scope for vafi#4) get tracked? It's flagged in
  workgraph.md and runtime DESIGN, but there's no observation
  filed against it. Should it be a `got_*` (gotcha) now or wait
  for a deployment shape that actually trips it?
- **Q5.** Where does drift-findings output (X3) live structurally
  in the SDD pipeline? The current `project-repo-DESIGN.md`
  doesn't have a place for it. Worth a follow-up decision.

## References

- This workgraph's workgraph.md and plan.md (same directory).
- `viloforge-platform/docs/implementation-roadmap-PLAN.md` §"Phase
  1 — Manual SDD validation" and §"step 8 Extract methodology".
- `viloforge-platform/docs/project-repo-DESIGN.md` §"Worked
  example" (now updated to kind: bugfix) and §"methodologies/<role>.md".
- `viloforge-platform/docs/vafi-runtime-DESIGN.md` §"Phase 5 —
  Draining" and §"The startup-reconciliation invariant" (now
  updated to as-built endpoint names).
- Commits: `ef48c8f` (platform doc fixes), `33cbcea` (project
  artifacts including this retro after amend).
- User memories applied during the session:
  `feedback_discuss_points_conversationally`,
  `feedback_verify_before_claiming`,
  `feedback_kb_recent_first_on_resume`,
  `feedback_no_ai_attribution_in_commits`,
  `feedback_proceed_per_protocol`.
