# Research and Investigation References

**Last reviewed:** 2026-10-05

This document is the external investigation map for the project. A future model or engineer should use it to recover the reasoning behind the design without needing the original conversation.

## Reading order

When taking over this project, read:

1. `AGENTS.md`
2. `docs/00-rationale.md`
3. `docs/01-design.md`
4. `docs/02-implementation-plan.md`
5. this document
6. `docs/04-webhmf-environment.md` once it exists

External projects are references, not dependencies by default. Re-check their current docs and licenses before implementation because these tools are moving quickly.

## Jev / TypeSafe

- TypeSafe home: https://typesafe.ai/
- Jev introduction: https://typesafe.ai/blog/introducing-system-one-models-and-jev
- Jev API / Swagger: https://api.typesafe.ai/docs
- Jev API / ReDoc: https://api.typesafe.ai/redoc
- Workflow evaluations: https://evals.typesafe.ai/

Investigate:
- Noul / Choice / Score question types,
- `/v1/systemone` request/response semantics,
- parallel typed questions,
- probability/calibration behavior,
- available model names,
- latency/cost limits,
- authentication and SDK/API options.

Project lesson:
Jev should answer narrow probabilistic questions. It should not own process permissions, budgets, workspace state, or irreversible execution.

## JevRouter

- Repository: https://github.com/BillionsBobby/JevRouter

Investigate:
- capability representation,
- typed route/plan decisions,
- availability and permission filtering,
- risk/confirmation policy around Jev,
- serial/batch/decomposition planning,
- benchmark scope and limitations.

Project lesson:
Borrow the idea that models, tools, CLIs, skills, and agents can all be capabilities while deterministic code owns availability and policy.

Do not treat routing benchmark performance as proof of end-to-end coding success.

## Foreman

- Repository: https://github.com/thruwire/foreman
- Why Jev fits Foreman: https://github.com/thruwire/foreman/blob/main/docs/why-jev.md
- Main runtime overview: https://github.com/thruwire/foreman/blob/main/README.md

Investigate:
- independent worker/supervisor loops,
- compact bounded observations,
- Jev responsibility checks,
- deterministic policy after Jev scoring,
- Codex App Server transport,
- OpenCode backend,
- steering versus stop/retry,
- verifier-worker behavior,
- event persistence,
- current single-worker limitation,
- roadmap for simultaneous workers.

Project lesson:
This is the strongest reference for the supervised single-worker path:
the coding agent works, Jev watches, deterministic policy decides whether to intervene.

## Claudexor

- Repository: https://github.com/razzant/claudexor

Investigate:
- isolated Best-of-N/race workspaces,
- Codex and OpenCode harness abstractions,
- cross-family reviewers,
- arbitration,
- bounded repair,
- deterministic gates,
- protected paths,
- usage/cost/quota facts.

Project lesson:
Borrow the hard-mode pattern:

```text
isolated candidates
  -> independent evidence
  -> review
  -> bounded repair
  -> arbitration
```

Do not import Claudexor's whole control plane into this project; it overlaps with the router/runtime responsibilities we already define.

## JevLoop

- Repository: https://github.com/parkavenue9639/jevloop
- Architecture: https://github.com/parkavenue9639/jevloop/blob/main/docs/architecture.md

Investigate:
- one durable ledger/transcript,
- compact Jev projection,
- richer LLM projection,
- deterministic runtime kernel,
- side-by-side Jev vs LLM-only experiments,
- state ownership and durability boundaries.

Project lesson:
Use one authoritative event history and derive purpose-specific projections from it. Neither Jev nor the generative model owns runtime truth.

JevLoop is pre-alpha; treat it as an architectural reference rather than a stable runtime dependency.

## Codex

Official references:

- Codex harness / App Server architecture:
  https://openai.com/index/unlocking-the-codex-harness/
- Codex App Server developer reference:
  https://developers.openai.com/siwc/token-sharing-open-source/codex-app-server

Investigate on WebHMF:
- installed Codex version,
- App Server availability,
- thread/turn lifecycle,
- steering/interruption,
- event notifications,
- model selection,
- exec vs App Server vs SDK trade-offs,
- authentication path actually in use.

Project lesson:
Prefer a structured lifecycle/control interface over terminal scraping, but do not hard-code App Server until the installed workstation version is inspected.

## OpenCode

- Main docs: https://opencode.ai/v2/docs
- CLI docs: https://opencode.ai/v2/docs/cli
- CLI command reference: https://opencode.ai/v2/docs/cli/commands/

Investigate on WebHMF:
- `opencode run --format json`,
- headless/background server API,
- ACP support,
- session continuation/forking,
- model/agent selection,
- cancellation/intervention semantics,
- permission configuration.

Project lesson:
Prefer JSON, ACP, or server/API interfaces when they provide a more reliable event boundary than parsing human-oriented terminal output.

## Pi Coding Agent

- Project website: https://pi.dev/
- SDK docs: https://pi.dev/docs/latest/sdk
- RPC overview: https://pi.dev/docs/latest/rpc
- RPC command reference: https://pi.dev/docs/latest/rpc-commands
- JSON event stream: https://pi.dev/docs/latest/json

Investigate on WebHMF:
- SDK versus RPC trade-off,
- session isolation,
- event stream semantics,
- steering while running,
- `agent_settled` lifecycle,
- model switching,
- provider/model discovery,
- cancellation/disposal,
- extension behavior.

Project lesson:
Pi is especially useful because harness and underlying model are separable. One Pi adapter may expose multiple worker configurations backed by different model families.

## Ecosystem indexes

- Awesome Agent Orchestrators:
  https://github.com/andyrewlee/awesome-agent-orchestrators
- Awesome Jev:
  https://github.com/cobanov/awesome-jev

Use these as discovery maps, not evidence that a listed project should be adopted.

Any project considered for code reuse still needs direct source, license, maintenance, and fit inspection.

## Test-time compute research

### Refining Over Resampling: Test-Time Self-Correction for LLM Reasoning

- arXiv: https://arxiv.org/abs/2608.05643

Investigate:
- diminishing returns from wider sampling,
- semantic diversity saturation,
- breadth versus depth allocation,
- critique/correction loops,
- verifier-free aggregation,
- benchmark configuration and limitations.

Project lesson:
Our v0 does not reproduce the paper. It takes one practical lesson: spend extra compute on refinement and genuinely different trajectories rather than blind repeated sampling.

### Broader topics to research before extending v0

Search current primary sources for:
- inference-time/test-time compute scaling,
- adaptive compute allocation,
- process and outcome reward models,
- self-correction limits,
- multi-agent debate failure modes,
- verifier reliability,
- coding-agent evaluation,
- cost-aware routing.

Do not turn a research result into a design invariant until the primary source itself has been read.

## Questions a takeover model should answer before changing architecture

Before adding a new orchestrator, reviewer, search strategy, or worker:

1. What concrete failure in the current minimal loop does this solve?
2. Can deterministic code or an existing verifier solve it more cheaply?
3. Does it improve information quality, or merely add another model opinion?
4. Does it add genuine diversity or just another resample?
5. What new state or side-effect authority does it require?
6. Can the decision be bounded enough for Jev?
7. Does the change preserve an inspectable reason for escalation?
8. How would run reports show that it helped?
9. Does WebHMF actually expose the integration surface being assumed?
10. Are licensing and credential boundaries understood?

## Canonical local authority

A future model should treat these files as project authority, in order:

1. `AGENTS.md` — takeover instructions and non-negotiable project rules.
2. `docs/00-rationale.md` — why this exists.
3. `docs/01-design.md` — architecture and responsibility boundaries.
4. `docs/02-implementation-plan.md` — concrete execution sequence.
5. `docs/03-research-references.md` — external investigation map.
6. `docs/04-webhmf-environment.md` — actual workstation facts once created.
7. Later decision records and experiment protocol documents.

If current external documentation conflicts with an old assumption in this repository, update the assumption explicitly rather than silently coding around it.
