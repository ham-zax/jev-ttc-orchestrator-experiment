# Jev-Gated Test-Time Compute Orchestrator Implementation Plan

**Goal:** Build a portable local experiment runner that adaptively routes coding tasks between direct, supervised, and two-candidate breadth execution using Jev for bounded decisions, Codex/OpenCode/Pi-compatible worker adapters for coding, deterministic policy for control flow, and objective verification when available.

**Architecture:** A TypeScript/Node.js control plane owns run state, budgets, worker lifecycle, workspace isolation, event history, and legal state transitions. Jev supplies typed routing and supervision signals but cannot execute actions directly. Coding harnesses sit behind a common worker interface; a strong reviewer is invoked only when semantic comparison or repair is required.

**Tech Stack:** TypeScript, Node.js, JSONL event storage, YAML configuration, Jev decision API/SDK behind an adapter, Codex/OpenCode/Pi worker adapters, Git-backed isolated candidate workspaces.

## Global Constraints

- Prefer Codex, OpenCode, and Pi Agent as worker harnesses.
- Do not require Claude Code.
- Keep harness identity separate from underlying model identity.
- Jev produces bounded signals; deterministic code owns side effects and state transitions.
- Objective verification outranks model opinion.
- Start with one worker by default and at most two candidates in breadth mode.
- Permit at most one repair pass and one retry per candidate in the first experiment.
- Do not implement MCTS, large Best-of-N sampling, learned PRMs, unbounded debate, or a general workflow engine.
- Keep every escalation and action decision inspectable from the durable run history.
- Keep credentials and machine-specific auth outside the repository.
- Keep the repository portable from WebHarness ARM to the actual WebHMF workstation.
- Do not assume target-workstation Codex/OpenCode/Pi invocation details until that workstation is inspected.
- This implementation plan does not add a test suite or test-running work; experiment evaluation is handled through recorded runs and observable acceptance criteria.

## Planned File Map

```text
README.md
package.json
tsconfig.json
.gitignore

config/
  example.yaml

src/
  cli/
    main.ts

  core/
    types.ts
    events.ts
    budgets.ts
    policy.ts
    errors.ts

  config/
    schema.ts
    load.ts

  decisions/
    provider.ts
    schemas.ts
    jev.ts

  routing/
    router.ts

  supervisor/
    supervisor.ts

  workers/
    adapter.ts
    registry.ts
    process-worker.ts
    codex.ts
    opencode.ts
    pi.ts

  verification/
    adapter.ts
    registry.ts
    command.ts

  review/
    reviewer.ts

  runtime/
    event-store.ts
    workspace.ts
    orchestrator.ts

  reporting/
    run-report.ts

docs/
  00-rationale.md
  01-design.md
  02-implementation-plan.md
  03-research-references.md          # external investigation and takeover map
  04-webhmf-environment.md           # created during WebHMF integration
  05-experiment-protocol.md          # created before comparative runs
```

---

### Task 1: Bootstrap the minimal TypeScript control-plane repository

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `.gitignore`
- Create: `config/example.yaml`
- Create: `src/cli/main.ts`
- Create: `src/core/errors.ts`

**Interfaces:**
- Consumes: the architecture constraints in `docs/01-design.md`
- Produces: a runnable local CLI entry point and repository-wide configuration conventions used by all later tasks

**Steps:**
- [ ] Configure the package as a private TypeScript project with Node.js as the runtime.
- [ ] Use a small dependency surface: schema validation plus YAML parsing; use Node built-ins for CLI parsing and subprocess control unless a concrete need appears.
- [ ] Define scripts for build and CLI execution without introducing a framework-specific runtime.
- [ ] Add ignored local paths for run artifacts, temporary workspaces, local configuration, credentials, and environment files.
- [ ] Create `config/example.yaml` with routing thresholds, budgets, worker descriptors, run-artifact location, and verification configuration; contain no real credentials or absolute machine paths.
- [ ] Create a CLI entry point that parses a task, optional execution-tier override, config path, and repository/workspace path, then delegates to an orchestrator interface that later tasks will supply.
- [ ] Define typed user-facing configuration and runtime errors rather than throwing anonymous strings.

**Acceptance criteria:**
- The repository has a small, understandable Node/TypeScript entry point.
- Example configuration expresses all current policy knobs without exposing secrets.
- No worker or Jev implementation details leak into CLI argument parsing.

---

### Task 2: Define authoritative run types, events, budgets, and legal policy actions

**Files:**
- Create: `src/core/types.ts`
- Create: `src/core/events.ts`
- Create: `src/core/budgets.ts`
- Create: `src/core/policy.ts`

**Interfaces:**
- Consumes: execution tiers and action vocabulary from `docs/01-design.md`
- Produces: shared types for task envelopes, worker observations, candidate state, run state, budgets, events, and deterministic runtime actions

**Steps:**
- [ ] Define `TaskEnvelope`, `ExecutionTier`, `CandidateId`, `RunId`, `WorkerDescriptor`, `VerificationResult`, and `ReviewerFinding`.
- [ ] Define the legal `RuntimeAction` union: `CONTINUE`, `VERIFY`, `STEER`, `RETRY`, `ESCALATE`, `START_SECOND_CANDIDATE`, `REVIEW`, `REPAIR`, `FINISH`, `FAIL`.
- [ ] Define immutable event payload types for run start, routing, worker lifecycle, observations, decision signals, arbiter actions, verification, review, repair, completion, and failure.
- [ ] Define `RunBudget` and `BudgetUsage` with limits for elapsed time, worker invocations, candidate count, Jev decisions, reviewer calls, retries, and repairs.
- [ ] Implement pure budget-check helpers that return allowed/denied plus a reason.
- [ ] Implement a pure deterministic policy function that accepts current state, signals, evidence, availability, and budget, and returns exactly one legal action plus a machine-readable reason.
- [ ] Make illegal transitions explicit errors rather than silent fallbacks.

**Acceptance criteria:**
- Later modules can express orchestration state without importing any provider-specific SDK.
- Every side-effecting action has a typed deterministic-policy path.
- Budget exhaustion always maps to an explicit finish/fail decision rather than an implicit loop exit.

---

### Task 3: Implement configuration loading and schema validation

**Files:**
- Create: `src/config/schema.ts`
- Create: `src/config/load.ts`
- Modify: `config/example.yaml`

**Interfaces:**
- Consumes: YAML configuration
- Produces: validated `OrchestratorConfig` used by routing, budgets, workers, verification, and reporting

**Steps:**
- [ ] Define the configuration schema for routing thresholds, budgets, worker descriptors, run-artifact directory, optional reviewer descriptor, and verification discovery policy.
- [ ] Reject unknown execution tiers, negative budgets, more than two configured breadth candidates for v0, and configurations that request a worker not present in the registry.
- [ ] Permit environment-variable references for secrets/provider endpoints without persisting resolved secret values into run events.
- [ ] Normalize relative paths from the repository root rather than relying on ARM-specific absolute paths.
- [ ] Make explicit CLI mode overrides take precedence over automatic routing while still recording the override in the run history.

**Acceptance criteria:**
- A validated configuration is environment-neutral.
- Machine paths and credentials are not required in committed configuration.
- v0 limits are enforced centrally rather than scattered across adapters.

---

### Task 4: Build the append-only event store and run projections

**Files:**
- Create: `src/runtime/event-store.ts`
- Modify: `src/core/events.ts`
- Modify: `src/core/types.ts`

**Interfaces:**
- Consumes: typed `RunEvent` objects
- Produces: append-only JSONL run history plus reconstructed runtime/Jev/reviewer projections

**Steps:**
- [ ] Store each run in its own artifact directory with one append-only JSONL event log.
- [ ] Assign monotonically increasing sequence numbers within a run.
- [ ] Implement reconstruction of authoritative `RunState` by folding events rather than maintaining an unrelated shadow state.
- [ ] Implement a compact Jev projection containing bounded factual state only.
- [ ] Implement a richer reviewer projection containing task, relevant worker output/diff summaries, verification results, and explicit constraints.
- [ ] Exclude credentials, raw auth material, and unrelated environment variables from all event payloads.
- [ ] Record the policy configuration snapshot or digest needed to explain historical routing decisions.

**Acceptance criteria:**
- A completed or interrupted run can be explained from its event history.
- Jev and reviewer contexts are projections of the same authoritative history rather than independent memories.
- Sensitive machine auth data is absent from persisted run artifacts.

---

### Task 5: Define Jev decision contracts without giving Jev execution authority

**Files:**
- Create: `src/decisions/provider.ts`
- Create: `src/decisions/schemas.ts`
- Create: `src/decisions/jev.ts`

**Interfaces:**
- Consumes: compact task/run projections
- Produces: validated `RoutingSignals` and `SupervisionSignals`

**Steps:**
- [ ] Define `DecisionProvider` with separate methods for routing and supervision signals.
- [ ] Define strict schemas for difficulty, breadth value, verification value, ambiguity, consequence, preferred capability IDs, stuck probability, progress quality, verification need, local-repair likelihood, escalation usefulness, and completion plausibility.
- [ ] Implement the Jev adapter so provider responses are parsed into those schemas and invalid/out-of-range values are rejected.
- [ ] Keep provider transport configuration injectable so WebHMF discovery can select the available Jev SDK/API mechanism without changing routing code.
- [ ] Implement a deterministic Jev-unavailable fallback result that cannot automatically trigger breadth mode.
- [ ] Record each decision request's non-secret input projection digest and parsed output as events.

**Acceptance criteria:**
- Jev cannot return a free-form executable command to the runtime.
- A malformed Jev response cannot directly cause a worker action.
- If Jev is unavailable, the system remains usable in a conservative deterministic mode.

---

### Task 6: Implement capability routing

**Files:**
- Create: `src/routing/router.ts`
- Modify: `src/core/policy.ts`
- Modify: `src/core/events.ts`

**Interfaces:**
- Consumes: task envelope, validated worker registry, routing signals, budget, explicit user overrides
- Produces: `DIRECT`, `SUPERVISED`, or `BREADTH` plus selected worker descriptor IDs and an inspectable reason

**Steps:**
- [ ] Normalize raw task input into the minimum routing context needed by Jev.
- [ ] Apply deterministic availability and policy constraints before model preference.
- [ ] Implement explicit override behavior for reproducible experiment runs.
- [ ] Route high-consequence tasks to at least supervised mode.
- [ ] Permit breadth only when a second compatible worker exists, the breadth signal crosses the configured threshold, and budget allows two candidates.
- [ ] Record both Jev signals and the deterministic routing decision separately.
- [ ] Avoid permanently assigning cognitive roles such as “critic” or “planner” to a vendor name.

**Acceptance criteria:**
- The same routing signals and configuration always produce the same route.
- Breadth cannot occur merely because two workers are installed.
- A route explains which signal, threshold, override, or availability rule caused it.

---

### Task 7: Implement the worker abstraction and registry

**Files:**
- Create: `src/workers/adapter.ts`
- Create: `src/workers/registry.ts`
- Create: `src/workers/process-worker.ts`

**Interfaces:**
- Consumes: worker descriptors and task/candidate inputs
- Produces: worker handles, structured observations, optional steering, termination, and final worker results

**Steps:**
- [ ] Define `WorkerAdapter`, `WorkerHandle`, `WorkerCapabilities`, `WorkerObservation`, and `WorkerResult`.
- [ ] Represent live steering, structured-event support, cancellation, and underlying-model metadata as capabilities rather than assumptions.
- [ ] Implement shared subprocess lifecycle utilities with explicit stdout/stderr capture, cancellation, exit status, and bounded output retention.
- [ ] Build a registry that resolves configured worker descriptor IDs to adapter implementations.
- [ ] Reject duplicate worker IDs and descriptors whose declared capabilities contradict their adapter.
- [ ] Ensure adapter output is converted into typed observations before it enters supervision.

**Acceptance criteria:**
- Runtime/orchestrator code does not know Codex/OpenCode/Pi command syntax.
- A worker that cannot steer can still participate through deterministic stop/retry semantics.
- The registry can hold multiple Pi descriptors backed by different underlying models.

---

### Task 8: Perform WebHMF worker/environment discovery before implementing real worker commands

**Files:**
- Create on the target workstation: `docs/04-webhmf-environment.md`
- Modify after discovery: `config/example.yaml` only if new portable capability fields are required

**Interfaces:**
- Consumes: actual WebHMF workstation installations and authenticated local agent environments
- Produces: a concrete adapter contract for Codex, OpenCode, and Pi based on observed interfaces rather than guessed invocation syntax

**Steps:**
- [ ] After the repository is copied/cloned to WebHMF, inspect installed Codex version and available structured interfaces, including whether App Server, exec, or another local integration is available.
- [ ] Inspect OpenCode version and its available process/API/event surfaces.
- [ ] Inspect Pi installation and whether RPC, SDK, or CLI mode is the narrowest stable integration.
- [ ] Record executable locations only as discovery evidence; do not commit user-specific absolute paths into portable defaults.
- [ ] Record each harness's cancellation semantics, structured-event support, live-steering support, workspace behavior, and model-selection mechanism.
- [ ] Record Jev client/credential availability without copying credentials into the document.
- [ ] Choose one primary integration surface per harness and one fallback only where observed behavior justifies it.

**Acceptance criteria:**
- `docs/04-webhmf-environment.md` contains enough observed detail to implement all three adapters without guessing.
- No provider token, cookie, credential cache, or secret value is added to Git.
- Adapter implementation begins only after this discovery artifact exists.

---

### Task 9: Implement Codex, OpenCode, and Pi adapters from workstation evidence

**Files:**
- Create: `src/workers/codex.ts`
- Create: `src/workers/opencode.ts`
- Create: `src/workers/pi.ts`
- Modify: `src/workers/registry.ts`
- Modify: `src/config/schema.ts`

**Interfaces:**
- Consumes: the exact integration surfaces recorded in `docs/04-webhmf-environment.md`
- Produces: three conforming `WorkerAdapter` implementations

**Steps:**
- [ ] Implement Codex using the richest stable structured interface discovered on WebHMF; expose live steering only if that interface actually supports it.
- [ ] Implement OpenCode using its observed structured/process interface and map its lifecycle to the common worker events.
- [ ] Implement Pi using the discovered RPC/SDK/CLI mechanism, preserving separation between Pi as a harness and its configured underlying model.
- [ ] Normalize completion, failure, tool/event, and cancellation behavior into shared observation types.
- [ ] Keep provider/model-specific flags in descriptor configuration rather than branching throughout orchestrator code.
- [ ] Surface unsupported requested capabilities as deterministic configuration errors.

**Acceptance criteria:**
- Codex, OpenCode, and Pi can each be selected by descriptor ID without changing orchestrator code.
- A worker's model can be changed through configuration where the harness supports it.
- Worker-specific process details remain isolated inside the adapter modules.

---

### Task 10: Implement workspace isolation for breadth candidates

**Files:**
- Create: `src/runtime/workspace.ts`
- Modify: `src/core/events.ts`
- Modify: `src/core/types.ts`

**Interfaces:**
- Consumes: source repository path and candidate ID
- Produces: isolated candidate workspace path and cleanup metadata

**Steps:**
- [ ] Define a workspace-provider interface so the runner can use Git worktrees or another safe local isolation method.
- [ ] For Git repositories, create candidate-specific worktrees/branches without allowing candidate A and B to mutate the same working tree.
- [ ] Record base revision, candidate workspace identity, and final changed-file summary in run events.
- [ ] Refuse breadth mode when safe isolation cannot be established.
- [ ] Keep cleanup explicit; do not silently delete a candidate workspace containing unreported changes after an abnormal stop.

**Acceptance criteria:**
- Two breadth candidates never share a mutable working tree.
- The final report can identify which candidate produced which diff.
- An interrupted breadth run does not destroy evidence needed for diagnosis.

---

### Task 11: Implement verification adapters and evidence precedence

**Files:**
- Create: `src/verification/adapter.ts`
- Create: `src/verification/registry.ts`
- Create: `src/verification/command.ts`
- Modify: `src/core/policy.ts`

**Interfaces:**
- Consumes: repository/task verification configuration and candidate workspace
- Produces: typed verification evidence with command/source, exit state, bounded output, and pass/fail/unknown status

**Steps:**
- [ ] Define a generic verification-adapter contract independent of worker identity.
- [ ] Implement an explicit command verifier for repository-provided commands configured by the user or discovered from authoritative repository instructions.
- [ ] Represent “no objective verifier available” as `unknown`, not success.
- [ ] Give deterministic passing/failing verification precedence over reviewer preference in the arbiter.
- [ ] Bound captured command output and retain a path/reference when full output must remain in run artifacts.
- [ ] Record each verifier invocation and result in the event stream.

**Acceptance criteria:**
- A model cannot overrule a deterministic failing verifier.
- Absence of verification is visible in the final status/report.
- Verification configuration is repository/task-specific rather than globally invented by the orchestrator.

---

### Task 12: Implement supervised single-worker execution

**Files:**
- Create: `src/supervisor/supervisor.ts`
- Create: `src/runtime/orchestrator.ts`
- Modify: `src/core/policy.ts`
- Modify: `src/runtime/event-store.ts`

**Interfaces:**
- Consumes: routed task, selected worker, Jev provider, verification registry, budgets
- Produces: an event-driven supervised run that terminates in finish/fail/escalate

**Steps:**
- [ ] Start the selected worker and emit worker lifecycle events.
- [ ] Create coarse observation checkpoints from worker completion claims, command failures, material diff changes, explicit agent pauses, or structured worker events.
- [ ] At eligible checkpoints, build the compact Jev projection and request supervision signals.
- [ ] Pass signals, current evidence, and budget into deterministic policy.
- [ ] Implement `CONTINUE`, `VERIFY`, `STEER`, `RETRY`, `ESCALATE`, `FINISH`, and `FAIL` transitions.
- [ ] Convert `STEER` to deterministic stop/retry behavior when the active adapter cannot live-steer.
- [ ] Enforce one retry per candidate and reject any retry request after budget exhaustion.
- [ ] Return the best verified state on budget stop and label incomplete/unverified outcomes accurately.

**Acceptance criteria:**
- A supervised run never loops solely because “more reasoning might help.”
- Every retry or escalation has an event-recorded evidence reason.
- Unsupported live steering degrades predictably rather than crashing the run.

---

### Task 13: Implement reviewer and localized repair

**Files:**
- Create: `src/review/reviewer.ts`
- Modify: `src/core/types.ts`
- Modify: `src/core/policy.ts`
- Modify: `src/runtime/orchestrator.ts`

**Interfaces:**
- Consumes: reviewer projection plus one or two candidate states
- Produces: structured keep/repair/reject findings, localized defect description, and preferred candidate when comparison is required

**Steps:**
- [ ] Define reviewer output as structured findings rather than unrestricted “pick a winner” prose.
- [ ] Require the reviewer to separate deterministic evidence from semantic judgment.
- [ ] Capture strongest assumption, weakest assumption, concrete defect, missing edge cases, and keep/repair/reject recommendation.
- [ ] Permit repair only when the arbiter confirms repair budget remains and the finding is localized.
- [ ] Send a repair instruction to the chosen worker that preserves valid work and targets the recorded defect.
- [ ] After repair, return to verification rather than automatically requesting another reviewer round.
- [ ] Enforce a maximum of one repair pass in v0.

**Acceptance criteria:**
- The runtime can explain exactly what defect triggered repair.
- A repair does not erase the prior candidate history.
- A failed repair cannot trigger an unbounded reviewer/repair loop.

---

### Task 14: Implement two-candidate breadth mode

**Files:**
- Modify: `src/runtime/orchestrator.ts`
- Modify: `src/runtime/workspace.ts`
- Modify: `src/review/reviewer.ts`
- Modify: `src/core/policy.ts`

**Interfaces:**
- Consumes: BREADTH route, two worker descriptors, isolated workspaces, verifier, reviewer, budget
- Produces: candidate A/B evidence and one final accepted/repaired/unresolved result

**Steps:**
- [ ] Create two isolated candidate workspaces from the same base revision.
- [ ] Start candidate A and B with intentionally distinct worker/model configurations or strategy instructions recorded in run state.
- [ ] Permit concurrent execution only when both adapters and the configured runtime budget safely support it; otherwise execute sequentially while preserving isolation.
- [ ] Run applicable deterministic verification independently for both candidates.
- [ ] If deterministic evidence clearly establishes one valid candidate and the other invalid, allow the arbiter to select without semantic voting.
- [ ] Otherwise invoke one reviewer comparison.
- [ ] Allow one localized repair of the selected candidate when justified.
- [ ] Re-run applicable deterministic verification after repair.
- [ ] Mark the run unresolved when evidence remains genuinely conflicting after allowed review/repair budget.

**Acceptance criteria:**
- Breadth mode never runs more than two candidates in v0.
- Candidate diversity is recorded explicitly.
- Deterministic evidence can short-circuit unnecessary reviewer work.
- Lack of consensus is allowed to remain unresolved instead of being hidden by forced voting.

---

### Task 15: Build run reporting and inspectable explanations

**Files:**
- Create: `src/reporting/run-report.ts`
- Modify: `src/cli/main.ts`
- Modify: `src/runtime/event-store.ts`

**Interfaces:**
- Consumes: completed or interrupted run event history
- Produces: concise terminal report plus machine-readable JSON summary

**Steps:**
- [ ] Report execution tier, selected worker configurations, final status, verification status, elapsed time, retry/repair/escalation counts, Jev-decision count, reviewer count, and available cost/token metadata.
- [ ] Include a chronological “why” summary for route, retry, escalation, breadth, repair, and finish decisions.
- [ ] Distinguish verified success, unverified completion, budget stop, policy stop, worker failure, and unresolved disagreement.
- [ ] Include candidate comparison metadata for breadth runs without dumping unnecessary private model reasoning.
- [ ] Print artifact paths for the event log and preserved candidate workspaces when relevant.

**Acceptance criteria:**
- A developer can answer “why did this become a two-agent run?” from the report without reading source code.
- A run never labels an unverified or unresolved state as verified success.
- Reports do not expose provider credentials or unrelated environment information.

---

### Task 16: Add explicit baseline and experiment execution modes

**Files:**
- Modify: `src/cli/main.ts`
- Modify: `src/runtime/orchestrator.ts`
- Modify: `src/reporting/run-report.ts`
- Create: `docs/05-experiment-protocol.md`

**Interfaces:**
- Consumes: a task corpus selected for the experiment and an explicit mode
- Produces: comparable run artifacts for single-worker baseline, supervised single-worker, and adaptive routing

**Steps:**
- [ ] Add explicit `baseline-single`, `supervised-single`, and `adaptive` modes that share worker and verification infrastructure.
- [ ] In the protocol document, define how the same task/repository starting state is reused across modes without allowing one run's mutations to contaminate another.
- [ ] Define the experiment's comparison fields: final status, verification status, worker calls, Jev calls, reviewer calls, retries, repairs, elapsed time, and available cost metadata.
- [ ] Require intentional task categories such as straightforward changes, ambiguous debugging, multi-file implementation, and tasks with objective verification.
- [ ] Document that the initial experiment is exploratory and should not claim universal model superiority from a small local sample.
- [ ] Keep model/harness versions in each run report so later comparisons remain interpretable.

**Acceptance criteria:**
- The same orchestrator can produce all three comparison modes.
- Experiment artifacts are comparable without needing to infer hidden differences in policy.
- The protocol distinguishes harness effects from underlying-model effects when configuration permits.

---

### Task 17: Update repository documentation after the first runnable experiment

**Files:**
- Create: `README.md` during bootstrap, then modify after first runnable experiment
- Modify: `docs/00-rationale.md`
- Modify: `docs/01-design.md`
- Modify: `docs/05-experiment-protocol.md`

**Interfaces:**
- Consumes: actual implemented behavior and first-run evidence
- Produces: documentation that describes the real system rather than the original assumptions

**Steps:**
- [ ] Make README show the actual execution modes and supported worker adapters.
- [ ] Record any design assumption invalidated by WebHMF discovery.
- [ ] Keep architectural deviations explicit rather than silently rewriting history.
- [ ] Document known limitations discovered during real Codex/OpenCode/Pi runs.
- [ ] Identify which next complexity, if any, is justified by evidence: better routing signals, richer verification, more robust steering, or broader search.

**Acceptance criteria:**
- Documentation matches the implemented control flow.
- Unsupported features are not presented as available.
- Any proposal for additional orchestration complexity is tied to a concrete observed limitation.

## Dependency Order

```text
Tasks 1-7
  portable control-plane foundations on the repository

Task 8
  WebHMF discovery boundary

Task 9
  real Codex/OpenCode/Pi adapters

Tasks 10-13
  workspace isolation, verification, supervision, repair

Task 14
  selective breadth

Tasks 15-16
  reporting and comparative experiment runner

Task 17
  evidence-based documentation update
```

## Initial Milestones

### Milestone A — Portable control plane

Tasks 1-7 complete.

Observable result: the repository contains the typed state machine, event history, routing policy, Jev decision contract, and worker abstraction without assuming workstation-specific command syntax.

### Milestone B — Real local workers

Tasks 8-9 complete on WebHMF.

Observable result: Codex, OpenCode, and Pi can each be launched through the same worker interface using the actual installed environment.

### Milestone C — Supervised single-worker experiment

Tasks 10-13 complete.

Observable result: one selected worker can be observed, verified, steered/retried, escalated, repaired once, and terminated through inspectable deterministic policy.

### Milestone D — Selective breadth experiment

Task 14 complete.

Observable result: a hard task can produce two isolated candidates, compare objective evidence, invoke one semantic reviewer when necessary, repair once, and return a verified or explicitly unresolved result.

### Milestone E — Comparative evidence

Tasks 15-17 complete.

Observable result: baseline-single, supervised-single, and adaptive modes produce comparable reports that show whether the extra orchestration is buying enough quality/recovery to justify its cost.

## Risk Register

### Routing overconfidence

A bounded decision model can still make bad classifications.

Mitigation: signals never execute actions directly; deterministic thresholds, availability, explicit mode overrides, and run history remain authoritative.

### Agent-interface drift

Codex, OpenCode, or Pi interfaces may differ from assumptions or change over time.

Mitigation: target-workstation discovery is a required implementation phase, and each harness is isolated behind a worker adapter.

### Orchestrator complexity overwhelms the benefit

It is easy to build a sophisticated controller whose overhead exceeds the quality gained.

Mitigation: DIRECT is the default cheap path; BREADTH is capped at two candidates; every new mechanism must be justified by experiment evidence.

### Correlated candidates

Two workers may still produce nearly identical approaches.

Mitigation: record the worker/model/strategy configuration for each candidate and intentionally diversify one dimension before considering more samples.

### Reviewer becomes another unverified authority

A strong frontier reviewer can still be wrong.

Mitigation: deterministic verification outranks reviewer preference; reviewer output is structured as findings and repair suggestions, not final truth.

### Workspace damage

Parallel agents can conflict if they mutate the same files.

Mitigation: breadth requires isolated workspaces and refuses to start when isolation cannot be established.

### Hidden cost growth

Repeated supervision or review can silently turn a cheap task into an expensive one.

Mitigation: explicit per-run budgets and reportable counts for every worker, Jev, reviewer, retry, repair, and escalation call.

## Completion Definition for the First Experiment

The first experiment is complete when:

1. one task can run in single-worker baseline mode;
2. the same task can run under supervised-single mode;
3. a hard task can be selectively routed into a two-candidate breadth run;
4. objective verification, when configured, determines success/failure ahead of model preference;
5. at most one repair and one retry per candidate are enforced;
6. each run produces an inspectable event history and report explaining routing and escalation;
7. Codex, OpenCode, and Pi are supported through the same worker contract on WebHMF;
8. no architecture component requires Claude Code;
9. secrets and workstation-specific auth remain outside Git;
10. the comparative report is sufficient to decide whether further test-time-compute complexity is justified.
