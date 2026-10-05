# Architecture and Experiment Design

## 1. Design objective

Build the smallest orchestration control plane that can answer this practical question:

> Can Jev-gated routing plus evidence-driven supervision and selective two-candidate breadth produce better coding outcomes than a single coding agent, without turning every task into an expensive multi-agent workflow?

The system is intentionally **not** a general autonomous-agent platform. It is an experiment runner with a clean path toward becoming a reusable local orchestrator if the experiment succeeds.

## 2. Design constraints

The first implementation must preserve these constraints:

- Prefer Codex, OpenCode, and Pi Agent as worker harnesses.
- Do not make Claude Code a required dependency.
- Keep worker harness choice separate from underlying model choice.
- Jev may recommend or score decisions but may not directly execute side effects.
- Deterministic policy owns budgets, retries, process control, workspace isolation, and stop conditions.
- Objective verification outranks model review.
- Default to one worker; use a second candidate only for named hard-mode reasons.
- Permit at most one repair pass in the first experiment.
- Avoid MCTS, large Best-of-N sampling, learned reward models, and unbounded debate.
- Keep every routing/escalation decision inspectable after a run.
- Make the control plane portable from WebHarness ARM to the actual WebHMF workstation.

## 3. Proposed technology shape

The preferred implementation language is **TypeScript on Node.js**.

Reasons:

- Pi exposes an embeddable TypeScript-friendly surface.
- Codex and OpenCode can be wrapped as subprocess or structured-service adapters.
- TypeScript makes typed bounded decisions and event schemas straightforward.
- The system can remain a small local CLI/service without introducing a heavyweight workflow engine.
- Worker invocation details can stay behind interfaces, allowing target-workstation discovery to determine the exact adapter implementation.

The design does **not** require a specific Jev SDK. Jev access is behind a `DecisionProvider` interface and can use an official SDK or direct API client available on the target workstation.

## 4. System overview

```text
                            USER TASK
                                |
                                v
                    +-----------------------+
                    |   Task Normalizer     |
                    +-----------+-----------+
                                |
                                v
                    +-----------------------+
                    | Jev Routing Signals   |
                    | difficulty            |
                    | breadth value         |
                    | verification value    |
                    | uncertainty / risk     |
                    +-----------+-----------+
                                |
                                v
                    +-----------------------+
                    | Deterministic Router  |
                    +-----------+-----------+
                                |
              +-----------------+------------------+
              |                 |                  |
              v                 v                  v
           DIRECT           SUPERVISED          BREADTH
        one worker          one worker          two isolated
        minimal loop        + supervision       candidates
              |                 |                  |
              +-----------------+------------------+
                                |
                                v
                    +-----------------------+
                    | Verification / Review |
                    +-----------+-----------+
                                |
                                v
                    +-----------------------+
                    | Deterministic Arbiter |
                    +-----------+-----------+
                                |
                                v
                           FINAL RESULT
```

## 5. Three execution tiers

### 5.1 DIRECT

Purpose: avoid orchestration overhead for straightforward tasks.

Flow:

```text
normalize
  -> select one worker configuration
  -> run
  -> collect final artifact/evidence
  -> perform only required deterministic verification
  -> finish
```

No continuous semantic supervision is required unless a hard failure occurs.

Typical reasons:

- low ambiguity,
- low failure consequence,
- short task,
- obvious worker fit,
- deterministic verification available.

### 5.2 SUPERVISED

Purpose: improve difficult single-worker execution without paying for a second full candidate.

Flow:

```text
normalize
  -> choose primary worker
  -> run to observation checkpoint
  -> collect evidence
  -> Jev supervision signals
  -> deterministic action
       CONTINUE
       VERIFY
       STEER
       RETRY
       ESCALATE
       FINISH
  -> repeat within budget
```

The supervisor should operate at **coarse checkpoints**, not after every token or tool call. Examples:

- worker asks for a decision,
- command fails,
- verification finishes,
- diff changes materially,
- worker reports completion,
- no progress occurs within a configured interval,
- or the runtime receives a structured worker event worth evaluating.

### 5.3 BREADTH

Purpose: spend extra compute only when independent approaches are likely to improve reliability.

Initial design:

```text
task
  -> isolate workspace A
  -> isolate workspace B
  -> candidate A using worker configuration A
  -> candidate B using worker configuration B
  -> run deterministic verification on both where possible
  -> compare evidence
  -> reviewer challenges both
  -> deterministic arbiter selects:
       accept A
       accept B
       repair A once
       repair B once
       unresolved
  -> verify repaired winner
  -> finish
```

The first experiment uses **two candidates only**.

Candidate diversity should be intentional. Examples:

- Codex vs OpenCode,
- Codex vs Pi with a different underlying model,
- same harness with meaningfully different model families,
- direct implementation vs edge-case/invariant-first instruction.

Breadth mode must not simply run the same prompt twice against identical configuration unless the experiment is explicitly measuring resampling.

## 6. Responsibility boundaries

### 6.1 TaskNormalizer

Consumes:

- raw user task,
- repository/workspace context,
- explicit user constraints,
- available worker capabilities.

Produces a typed `TaskEnvelope` containing only information needed for routing and execution.

It does not choose a worker.

### 6.2 DecisionProvider

Represents Jev-backed bounded decisions.

Candidate routing signals:

```ts
type RoutingSignals = {
  difficulty: 0 | 1 | 2 | 3 | 4;
  breadthValue: number;          // 0..1
  verificationValue: number;     // 0..1
  ambiguity: number;             // 0..1
  consequence: "low" | "medium" | "high";
  preferredCapabilityIds: string[];
};
```

Candidate supervision signals:

```ts
type SupervisionSignals = {
  appearsStuck: number;          // 0..1
  progressQuality: number;       // 0..1
  needsVerification: number;     // 0..1
  localRepairLikely: number;     // 0..1
  escalationUseful: number;      // 0..1
  completionPlausible: number;   // 0..1
};
```

These are **signals**, not actions.

### 6.3 Deterministic Router

Maps routing signals plus configuration and availability to one of:

```text
DIRECT
SUPERVISED
BREADTH
```

Example policy shape:

```text
if explicit mode override:
    use override

else if high consequence:
    at least SUPERVISED

else if breadthValue >= breadth threshold
        and second worker is available
        and budget permits:
    BREADTH

else if difficulty >= supervision threshold:
    SUPERVISED

else:
    DIRECT
```

Thresholds belong in configuration and are recorded in the run.

### 6.4 WorkerAdapter

All coding harnesses implement one conceptual contract.

```ts
interface WorkerAdapter {
  id: string;
  capabilities(): Promise<WorkerCapabilities>;
  start(input: WorkerStartInput): Promise<WorkerHandle>;
  observe(handle: WorkerHandle): Promise<WorkerObservation>;
  steer?(handle: WorkerHandle, instruction: string): Promise<void>;
  terminate(handle: WorkerHandle): Promise<void>;
}
```

The exact implementation can vary:

- Codex App Server or structured Codex process,
- Codex exec fallback,
- OpenCode process/API,
- Pi RPC/SDK process.

Live steering is optional. If unavailable, deterministic policy converts a steering need into a stop-and-retry action when allowed.

### 6.5 VerificationAdapter

Verification is intentionally independent of worker identity.

```ts
interface VerificationAdapter {
  id: string;
  supports(context: VerificationContext): Promise<boolean>;
  run(context: VerificationContext): Promise<VerificationResult>;
}
```

Examples:

- repository command,
- compiler,
- type checker,
- structured diff check,
- task-specific executable check.

The first implementation should not invent validation commands. It should discover or accept explicitly configured verification for the target repository.

### 6.6 Reviewer

A reviewer is a strong generative model/agent role used only when semantic critique is required.

Responsibilities:

- compare candidate assumptions,
- find concrete logic/implementation flaws,
- identify missing edge cases,
- recommend keep/repair/reject,
- and localize the smallest repair.

The reviewer does not override deterministic verification.

A reviewer may use Codex, OpenCode, Pi, or another configured frontier worker. The architecture does not permanently bind this role to a vendor.

### 6.7 Deterministic Arbiter

The arbiter owns legal state transitions.

Possible actions:

```ts
type RuntimeAction =
  | "CONTINUE"
  | "VERIFY"
  | "STEER"
  | "RETRY"
  | "ESCALATE"
  | "START_SECOND_CANDIDATE"
  | "REVIEW"
  | "REPAIR"
  | "FINISH"
  | "FAIL";
```

The arbiter consumes:

- current run state,
- budget,
- worker availability,
- Jev signals,
- verification results,
- reviewer findings,
- retry/repair counters,
- user policy.

It produces one legal action and a machine-readable reason.

## 7. Event-sourced run history

Every meaningful observation and decision should be recorded as an append-only event.

Conceptual event examples:

```text
run.started
routing.signals_received
routing.decision_made
worker.started
worker.observation
verification.started
verification.completed
supervision.signals_received
arbiter.action_selected
worker.steered
worker.retried
breadth.candidate_started
review.completed
repair.started
run.finished
run.failed
```

Each event contains:

- run ID,
- sequence number,
- timestamp,
- event type,
- source,
- relevant typed payload,
- optional parent/candidate ID.

The event history is the source for post-run explanation:

> Why did the system call a second worker?

The answer should be reconstructable from recorded routing signals, thresholds, availability, budget, and arbiter decision.

## 8. State projections

Borrowing the strongest idea from JevLoop, different consumers should receive different projections from the same run history.

### Jev projection

Small, bounded, structured, and stripped of unnecessary prose.

Examples:

- task category,
- latest worker status,
- verification status,
- changed-file count,
- recent failure summary,
- retry count,
- elapsed/budget fraction.

### Reviewer projection

Richer semantic context:

- original task,
- candidate answer/diff,
- relevant execution output,
- verification failures,
- explicit constraints.

### Runtime projection

Authoritative machine state:

- active processes,
- workspace locations,
- budgets,
- candidate IDs,
- counters,
- allowed next actions.

No model owns runtime truth.

## 9. Budgets

The first implementation should use simple hard budgets.

Per run:

- maximum wall-clock duration,
- maximum worker invocations,
- maximum Jev decisions,
- maximum reviewer calls,
- maximum retries,
- maximum repair passes,
- maximum concurrent candidates.

Initial experimental defaults should favor simplicity:

```text
candidates:      max 2
repair passes:   max 1
retry passes:    max 1 per candidate
review rounds:   max 1 before repair
```

Exact numeric time/token/cost thresholds should be configurable rather than hard-coded in the architecture document.

## 10. Stop conditions

A run should stop when any of these deterministic conditions is reached:

### Successful stop

- required deterministic verification passes, or
- no deterministic verifier exists and the configured completion policy is satisfied after review.

### Budget stop

- wall-clock budget exceeded,
- worker-call budget exceeded,
- reviewer budget exceeded,
- or allowed retry/repair count exhausted.

### Policy stop

- user cancellation,
- forbidden action,
- unavailable required capability,
- unrecoverable worker failure.

The runtime should return the **best verified state reached**, not silently pretend full success after a budget stop.

## 11. Local repair policy

Repair is preferred when:

- verification exposes a localized failure,
- reviewer identifies a specific defect,
- the rest of the candidate remains sound,
- and repair budget remains.

Repair is rejected when:

- the approach is fundamentally incompatible with the task,
- the candidate violates major architectural constraints,
- the worker has not produced a coherent state worth preserving,
- or a previous repair already failed.

The first experiment permits one repair pass.

## 12. Worker selection strategy

Worker identities are capabilities, not permanent cognitive roles.

Example registry:

```ts
type WorkerDescriptor = {
  id: string;
  harness: "codex" | "opencode" | "pi" | string;
  model?: string;
  supportsSteering: boolean;
  supportsStructuredEvents: boolean;
  costClass: "cheap" | "standard" | "frontier";
  tags: string[];
};
```

Possible registered configurations:

```text
codex-primary
opencode-independent
pi-cheap-workhorse
pi-frontier-reviewer
```

The same Pi harness may appear multiple times with different underlying models.

## 13. Failure and fallback behavior

### Jev unavailable

Fallback to deterministic conservative routing:

- straightforward explicit tasks -> DIRECT,
- otherwise -> SUPERVISED,
- never enter BREADTH automatically without a decision signal or explicit override.

### Primary worker unavailable

Choose the next compatible worker descriptor according to deterministic availability policy.

### Reviewer unavailable

Do not synthesize fake consensus. Use verification evidence and mark unresolved semantic disagreement explicitly.

### Verification unavailable

Record that no objective verifier was available. High-consequence tasks may require reviewer escalation or return an unresolved status depending on policy.

## 14. Configuration

Configuration should be declarative and environment-neutral.

Conceptual sections:

```yaml
routing:
  supervisionDifficultyThreshold: 2
  breadthValueThreshold: 0.75

budgets:
  maxCandidates: 2
  maxRetriesPerCandidate: 1
  maxRepairsPerRun: 1
  maxReviewerCalls: 1

workers:
  - id: codex-primary
    adapter: codex
  - id: opencode-independent
    adapter: opencode
  - id: pi-workhorse
    adapter: pi

verification:
  discovery: repository
```

Secrets, tokens, provider credentials, and machine-specific auth state are never written into this file.

## 15. Experiment modes

The runner should make comparison explicit.

### Baseline A — Single worker

```text
task -> one worker -> verification -> result
```

### Baseline B — Supervised single worker

```text
task -> worker -> Jev/evidence supervision -> repair/retry if justified -> verification
```

### Experiment C — Adaptive policy

```text
task -> Jev routing -> DIRECT / SUPERVISED / BREADTH
```

This allows us to ask whether complexity in the controller produces measurable benefit.

## 16. What to measure

Each run should emit enough metadata to compare:

- final status,
- verification status,
- worker configuration(s),
- number of worker invocations,
- number of Jev decisions,
- number of reviewer calls,
- retry count,
- repair count,
- escalation count,
- elapsed time,
- available token/cost metadata when providers expose it,
- and the deterministic reason for every action transition.

The experiment should not depend on provider-reported “confidence.”

## 17. Repository layout target

Planned structure:

```text
jev-ttc-orchestrator-experiment/
  README.md
  docs/
    00-rationale.md
    01-design.md
    02-implementation-plan.md
  src/
    cli/
    core/
      events.ts
      state.ts
      policy.ts
      budgets.ts
    decisions/
      provider.ts
      jev.ts
      schemas.ts
    routing/
      router.ts
    supervisor/
      supervisor.ts
    workers/
      adapter.ts
      registry.ts
      codex.ts
      opencode.ts
      pi.ts
    verification/
      adapter.ts
      registry.ts
    review/
      reviewer.ts
    runtime/
      orchestrator.ts
      workspace.ts
      event-store.ts
    reporting/
      run-report.ts
  config/
    example.yaml
```

This layout is a design target, not yet implemented.

## 18. WebHarness ARM -> WebHMF workstation handoff

This phase creates the specification and implementation plan on WebHarness ARM.

The next environment phase should occur on the actual WebHMF workstation and begin with **discovery, not assumptions**:

1. inspect installed Codex interfaces and versions,
2. inspect OpenCode interfaces and versions,
3. inspect Pi installation/API/RPC availability,
4. inspect Jev credentials/client availability,
5. identify local repository/workspace conventions,
6. identify how each harness exposes structured events and cancellation,
7. choose the narrowest adapter for each worker,
8. populate local, untracked configuration.

The repository should be transferred through normal Git clone/push/pull or another explicit copy mechanism chosen at that time.

No machine credentials, auth caches, or workstation-specific secrets should be copied from ARM.

## 19. Explicit non-goals for v0

The first implementation will not include:

- MCTS,
- beam-search reasoning trees,
- more than two simultaneous candidate trajectories,
- learned PRMs,
- autonomous long-term memory,
- distributed queues,
- multi-user service hosting,
- web UI,
- arbitrary plugin marketplace,
- automatic provider credential management,
- or self-modifying routing policy.

These can be considered only after the minimal experiment produces evidence that they are necessary.

## 20. Design decision summary

The proposed system combines four ideas without importing four competing orchestrators:

- **JevRouter idea:** bounded capability routing.
- **Foreman idea:** continuous evidence-driven supervision with deterministic action policy.
- **Claudexor idea:** isolated two-candidate hard mode with review and bounded repair.
- **JevLoop idea:** one authoritative run history with narrow projections and deterministic runtime ownership.

The result is intentionally smaller than any attempt to stack all four products:

```text
Jev decides bounded questions.
Workers do coding.
Reviewers do deep semantic critique.
Tools verify objective claims.
Deterministic code owns control flow.
```
