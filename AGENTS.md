# AGENTS.md — Project Handoff Instructions

## Purpose

This repository defines an experiment for a **Jev-gated test-time-compute coding orchestrator**.

The project is trying to answer one practical question:

> Can cheap bounded routing + evidence-driven supervision + selective two-candidate breadth produce better coding-agent outcomes than a conventional single-agent run without creating a research-grade orchestration system?

Do not expand the project into a general agent platform unless experiment evidence demonstrates a concrete need.

## Current status

The repository is in the **planning-only phase**.

No orchestration runtime has been implemented yet.

The planning workspace is WebHarness ARM:

```text
/home/ubuntu/repo/jev-ttc-orchestrator-experiment
```

The intended implementation workstation is the user's actual **WebHMF** environment. Do not guess WebHMF tool versions, paths, auth mechanisms, or process interfaces.

## Required reading order before implementation

Read these completely before writing runtime code:

1. `docs/00-rationale.md`
2. `docs/01-design.md`
3. `docs/02-implementation-plan.md`
4. `docs/03-research-references.md`

On WebHMF, create and then treat as factual environment authority:

5. `docs/04-webhmf-environment.md`

Before comparative runs:

6. `docs/05-experiment-protocol.md`

## Non-negotiable architecture rules

1. **Jev proposes bounded signals; deterministic code owns side effects.**
2. **Objective verification outranks model opinion.**
3. **One worker is the default. Extra compute requires a named reason.**
4. **v0 breadth is at most two isolated candidates.**
5. **Prefer one local repair before discarding a salvageable trajectory.**
6. **Retry is evidence-gated, not a blind loop.**
7. **Worker harness and underlying model are separate identities.**
8. **Codex, OpenCode, and Pi are preferred workers. Claude Code is optional.**
9. **One append-only run history is the source for state and post-run explanation.**
10. **No provider credential, token, cookie, auth cache, or workstation secret belongs in Git.**
11. **Do not claim sandboxing unless a real isolation boundary exists.**
12. **Do not add MCTS, PRMs, broad beam search, or large Best-of-N to v0.**

## External investigation map

Use `docs/03-research-references.md`.

Primary architectural references:

- Jev / TypeSafe — bounded structured decisions
- JevRouter — capability routing
- Foreman — Jev-supervised Codex/OpenCode workers
- Claudexor — isolated races / cross-model review / repair
- JevLoop — deterministic runtime and projection boundaries
- Codex official App Server/harness docs
- OpenCode official CLI/server/ACP docs
- Pi official SDK/RPC/event docs
- arXiv 2608.05643 — refinement over blind resampling

Do not rely on social-media summaries when the primary source is available.

## WebHMF implementation gate

Before implementing any real worker adapter on WebHMF, inspect and record:

- installed `codex` version and structured interfaces,
- installed `opencode` version and structured interfaces,
- installed `pi` version and SDK/RPC support,
- Jev API/SDK/auth availability,
- cancellation semantics,
- steering semantics,
- structured event surfaces,
- session/workspace behavior,
- model-selection mechanisms.

Write only non-secret findings to:

`docs/04-webhmf-environment.md`

Do not begin Codex/OpenCode/Pi adapter implementation until this file exists.

## Preferred implementation sequence

Follow `docs/02-implementation-plan.md`.

The first runnable milestone should be smaller than the eventual architecture:

```text
one task
  -> Jev route
  -> one real worker
  -> append-only events
  -> one configured verifier
  -> one supervision checkpoint
  -> deterministic finish/verify/retry decision
  -> run report
```

Only after that path is inspectable should you add the other worker adapters, then two-candidate breadth.

## Design discipline

When proposing a new subsystem, answer:

- What concrete failure does it solve?
- Could deterministic code solve it more cheaply?
- Is this genuine new information or another model opinion?
- How will run reports show whether it helped?
- Does it introduce new side-effect authority?
- Does it preserve inspectable reasons for actions?

If the answer is unclear, keep the feature out of v0.

## Documentation discipline

If implementation evidence invalidates a design assumption:

1. update `docs/04-webhmf-environment.md` with observed facts,
2. update `docs/01-design.md` if the architecture changes,
3. update `docs/02-implementation-plan.md` if task sequencing/interfaces change,
4. explain the change rather than silently diverging.

## Source reuse

The external projects are **references, not dependencies by default**.

Before copying or adapting source code:

- inspect the exact current license,
- record whether usage is inspiration, dependency, or adapted code,
- preserve required attribution,
- avoid importing an entire control plane when only one pattern is needed.

## Experiment principle

The project succeeds only if it can compare added orchestration against simpler baselines.

The core comparison is:

```text
single worker
vs
supervised single worker
vs
adaptive direct/supervised/breadth
```

More agent activity is not itself a success metric.
