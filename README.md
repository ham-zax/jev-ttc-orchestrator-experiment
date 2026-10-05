# Jev-Gated Test-Time Compute Orchestrator

Planning repository for a small, practical multi-agent coding experiment.

## Experiment question

Can a lightweight orchestration layer improve on a single coding agent by using:

- Jev for cheap bounded routing and supervision signals,
- deterministic code for policy and side effects,
- Codex / OpenCode / Pi as coding workers,
- objective verification when available,
- one local repair before restart,
- and a two-candidate hard mode only when breadth is worth the cost?

## Current status

**Planning complete; implementation has not started.**

This repository was created on WebHarness ARM. The next implementation phase is designed to move to the actual WebHMF workstation, inspect the real installed Codex/OpenCode/Pi interfaces, and implement adapters from observed capabilities rather than assumptions.

## Read in order

1. [Takeover and implementation rules](AGENTS.md)
2. [Why this exists and what we are testing](docs/00-rationale.md)
3. [Architecture and experiment design](docs/01-design.md)
4. [Concrete implementation plan](docs/02-implementation-plan.md)
5. [Research and investigation references](docs/03-research-references.md)

After moving to WebHMF, create `docs/04-webhmf-environment.md` from observed workstation facts before implementing real worker adapters.

## Intended control loop

```text
task
  -> Jev routing signals
  -> deterministic route
       DIRECT
       SUPERVISED
       BREADTH (max two candidates)
  -> Codex / OpenCode / Pi worker(s)
  -> evidence / verification
  -> Jev supervision signals where useful
  -> deterministic continue / verify / steer / retry / repair / escalate / finish
  -> inspectable run report
```

## Architectural rule

> Jev decides bounded questions. Workers do coding. Reviewers do deep semantic critique. Tools verify objective claims. Deterministic code owns control flow.

## Source inspirations

This experiment borrows ideas, not entire control planes, from:

- [JevRouter](https://github.com/BillionsBobby/JevRouter)
- [Foreman](https://github.com/thruwire/foreman)
- [Claudexor](https://github.com/razzant/claudexor)
- [JevLoop](https://github.com/parkavenue9639/jevloop)
- [awesome-agent-orchestrators](https://github.com/andyrewlee/awesome-agent-orchestrators)
- [awesome-jev](https://github.com/cobanov/awesome-jev)

See `docs/03-research-references.md` for the canonical URLs, primary-source research links, worker documentation, and the exact questions a future model should investigate before changing the design.
