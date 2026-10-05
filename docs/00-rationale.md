# Jev-Gated Test-Time Compute Orchestrator Experiment

## Status

Planning-only repository. No orchestration runtime has been implemented yet.

## Why this experiment exists

Modern coding agents are already strong enough to solve many tasks in one pass, but a single fixed execution policy wastes capability in two opposite ways:

1. **Easy work is over-served.** A frontier model may spend expensive reasoning tokens on routing, triage, retries, and obvious decisions that could have been handled cheaply.
2. **Hard work is under-served.** A single model can commit early to one approach, repeat the same failure mode on retries, or stop without an independent challenge or objective verification.

Research on test-time compute suggests that additional inference compute can improve difficult-task performance when it is allocated intelligently. The practical problem is that research-grade techniques such as large Best-of-N sampling, MCTS, process reward models, and broad evaluation sweeps are too expensive and operationally complex for a normal developer workflow.

This experiment asks a narrower engineering question:

> Can a small, understandable orchestration layer produce meaningfully better coding-agent outcomes than a single-agent baseline by spending extra compute only when evidence says it is useful?

The working hypothesis is that most of the practical benefit can come from a small number of mechanisms:

- cheap bounded routing,
- one strong primary coding worker,
- evidence-driven supervision,
- selective verification,
- one repair before restart,
- and a two-candidate hard mode only when genuine breadth is useful.

The goal is **not** to reproduce an academic search engine. The goal is to discover the simplest orchestration policy that is materially better than today's single-prompt / single-agent default.

## Core idea

The experiment separates four responsibilities that are often collapsed into one LLM call:

### 1. Bounded decision layer

Jev answers narrow questions such as:

- Which execution tier should this task use?
- Is a second independent approach likely to help?
- Does the current worker appear stuck?
- Is the available evidence sufficient to verify?
- Should the runtime continue, repair, retry, escalate, or stop?

Jev is never treated as the source of truth. It produces signals or bounded choices.

### 2. Coding-agent layer

Real coding work is performed by mature agent harnesses, with an initial preference for:

- Codex,
- OpenCode,
- Pi Agent.

Claude Code may be supported later, but it is not a design dependency.

The orchestrator distinguishes the **agent harness** from the **underlying model**. For example, Pi can be a worker harness while the actual model behind it is separately configurable.

### 3. Deterministic runtime and policy layer

Ordinary code owns:

- allowed actions,
- budgets,
- retry limits,
- process lifecycle,
- workspace isolation,
- state transitions,
- tool permissions,
- evidence collection,
- and final stop conditions.

Models propose; the runtime decides whether a proposal is allowed to happen.

### 4. Verification layer

When objective evidence is available, it outranks model opinion.

Examples include:

- compiler/type-checker results,
- repository-defined checks,
- deterministic commands,
- diff inspection,
- structured tool outputs,
- and other task-specific verification.

A model saying “this looks correct” is not verification.

## The minimal execution policy

The intended first experiment is deliberately small.

### Normal mode

```text
task
  -> Jev route / budget decision
  -> one primary worker
  -> collect evidence
  -> Jev supervision signals
  -> deterministic arbiter
       -> continue
       -> verify
       -> local repair / steer
       -> retry
       -> escalate
       -> finish
  -> final result
```

### Hard mode

Only when the router decides that meaningful diversity is worth the cost:

```text
task
  -> candidate A in isolated workspace
  -> candidate B in isolated workspace
  -> compare evidence and outputs
  -> strong reviewer / critic
  -> repair the stronger candidate when possible
  -> deterministic verification
  -> final result
```

The first hard-mode target is **two candidates**, not 8, 16, or 32.

## Why these design choices

### Why Jev for routing and supervision?

The orchestration problem contains many frequent, bounded decisions. Spending a frontier-model call on every “continue?”, “stuck?”, “verify?”, or “escalate?” decision defeats the economic purpose of adaptive test-time compute.

Jev is interesting here because the desired outputs are constrained decisions rather than free-form prose. Its role is therefore closer to a fast control signal than to a reasoning agent.

### Why not let Jev directly control processes?

A probabilistic decision model should not own irreversible control flow. A deterministic arbiter should apply thresholds, budgets, permissions, and safety policy before any action is executed.

This also makes the system easier to debug: we can inspect the signal separately from the policy that acted on it.

### Why start from Foreman-like supervision?

Foreman demonstrates a useful separation:

- observe a worker,
- derive bounded semantic checks,
- let deterministic policy choose actions.

It is also already aligned with the preferred worker ecosystem: Codex is a natural primary worker and OpenCode is a first-class alternative.

The experiment will borrow this control-loop shape rather than blindly forking Foreman's entire product surface.

### Why borrow from JevRouter?

JevRouter provides the cleanest front-door abstraction for choosing among capabilities rather than hard-coding one model for every task.

The experiment borrows the idea that models, agents, tools, and execution strategies can be represented as candidate capabilities, while permissions and availability remain deterministic.

### Why borrow from Claudexor?

Claudexor is most relevant to the hard-task path:

- isolated candidates,
- independent/cross-family review,
- bounded repair,
- arbitration.

We want that pattern, but only as an escalation mode. It should not be the default cost of every task.

### Why borrow from JevLoop?

JevLoop's strongest contribution is architectural discipline:

- one durable event/transcript source,
- projections for different decision makers,
- and a deterministic runtime kernel that owns execution truth.

We want this boundary even if we do not use JevLoop as the runtime itself.

## What this experiment is not

This repository is not intended to become:

- a general-purpose agent framework on day one,
- a replacement for Codex, OpenCode, or Pi,
- a 32-rollout benchmark harness,
- an MCTS implementation,
- a learned reward-model project,
- a universal “AI committee,”
- or a prompt-only orchestration trick.

The experiment should remain small enough that one developer can understand the full control flow.

## Initial hypotheses

### H1 — Cheap routing beats always-frontier orchestration

A bounded router can avoid wasting frontier-model calls on obvious tasks without materially reducing successful outcomes.

### H2 — Evidence-gated supervision beats blind retry

Retries should happen because the worker is demonstrably stuck, verification failed, or a concrete flaw was found—not because a fixed loop counter says to “try again.”

### H3 — Local repair beats full regeneration

When the current trajectory is mostly good, steering or repairing the local failure should be cheaper and more stable than discarding all work and starting over.

### H4 — Two genuinely different candidates are enough to test breadth value

For difficult tasks, two isolated candidates using different worker/model configurations should reveal whether selective breadth is useful before we consider larger sampling.

### H5 — Harness diversity may matter as much as model diversity

Codex, OpenCode, and Pi differ not only by the models they can call, but also by tool use, context management, process semantics, and interaction patterns. The experiment should measure the whole worker configuration rather than assume “model name” is the only variable.

### H6 — Deterministic evidence should dominate consensus

If tests, compilers, repository checks, or other deterministic evidence can decide correctness, model voting should never overrule them.

## Success criteria

The experiment is promising if it can demonstrate, on realistic coding tasks, that an adaptive policy can improve one or more of these without unacceptable regressions in the others:

- task success rate,
- recovery from a bad initial trajectory,
- frequency of unnecessary retries,
- frontier-model usage,
- total model cost,
- wall-clock time,
- number of destructive/invalid actions,
- and explainability of “why the orchestrator did that.”

The first milestone does **not** require proving a universal benchmark win. It requires making the control loop observable enough that we can compare:

1. single worker,
2. supervised single worker,
3. selectively escalated hard mode.

## Preferred worker strategy

Initial preference:

1. **Codex** — primary strong coding worker.
2. **OpenCode** — independent alternative worker / hard-mode candidate.
3. **Pi Agent** — lightweight, embeddable worker and future model-agnostic adapter.
4. **Claude Code** — optional future backend, not required for the architecture.

The system should never encode assumptions such as “Codex is always the planner” or “OpenCode is always the critic.” Roles are assigned by capability and policy, not permanently by brand.

## Portability and workstation boundary

This repository is being created on the WebHarness ARM environment under:

```text
/home/ubuntu/repo/jev-ttc-orchestrator-experiment
```

The ARM environment is the design and planning workspace for this phase.

A later phase will copy or clone the repository to the user's actual WebHMF workstation, where the real local agent environment can be inspected and adapters can be implemented against installed Codex, OpenCode, Pi, or other available harnesses.

Therefore:

- planning must not assume ARM-specific absolute paths,
- worker adapters must be configurable,
- process invocation must be isolated behind interfaces,
- provider credentials must never be stored in the repository,
- and environment discovery must happen on the target workstation rather than being guessed here.

## Architectural principles

1. **Models propose; deterministic code disposes.**
2. **Spend extra compute only for a named reason.**
3. **Prefer evidence over confidence language.**
4. **Repair before restart when the trajectory is salvageable.**
5. **Use diversity intentionally, not as blind resampling.**
6. **Keep worker harness and underlying model separate.**
7. **Keep the event history inspectable.**
8. **Make every escalation explainable.**
9. **Do not couple the design to Claude.**
10. **Start with the smallest architecture that can falsify the hypotheses.**

## Source projects studied

The experiment is informed by architectural ideas from:

- JevRouter — https://github.com/BillionsBobby/JevRouter
- Foreman — https://github.com/thruwire/foreman
- Claudexor — https://github.com/razzant/claudexor
- JevLoop — https://github.com/parkavenue9639/jevloop
- awesome-agent-orchestrators — https://github.com/andyrewlee/awesome-agent-orchestrators
- awesome-jev — https://github.com/cobanov/awesome-jev

These projects are references, not dependencies by default. Any code reuse must be evaluated separately for licensing, maintenance cost, and fit.
