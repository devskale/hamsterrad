# Hamsterrad — Looping Tooling for Pi
*For putting agents into the hamsterrad.*

<p align="center"><img src="media/logo.png" width="128" alt="Hamsterrad logo" /></p>

Agentic loop primitives for the [pi coding agent](https://pi.dev). Goal loops, iteration loops, for-each loops, research loops — composable extensions that turn pi into an autonomous, self-driving engineering system.

> **Status.** Early development. Research complete. Implementation beginning.
>
> *"You shouldn't be prompting coding agents anymore. You should be designing loops that prompt your agents."* — [Peter Steinberger](https://x.com/steipete/status/2063697162748260627)

## Why

Pi has no native loop mode. Claude Code has `/goal` and `/loop`. Codex has Automations and `/goal`. Pi has **nothing built-in** — but the most composable extension API of any harness.

This project fills that gap.

## What Exists (and why we're building anyway)

| Package | Shape | Gap |
|---------|-------|-----|
| [pi-goal-x](https://pi.dev/packages/pi-goal-x) | Goal mode | Heavy, no for-each, no batch |
| [pi-ralph](https://github.com/samfoy/pi-ralph) | Hat-based orchestration | Complex, not a general primitive |
| [pi-autoresearch](https://github.com/davebcn87/pi-autoresearch) | Experiment loop | Domain-specific |
| [rpiv-pi](https://github.com/juicesharp/rpiv-pi) | Ship-loop (5-skill pipeline) | Not a loop framework |

**None compose.** Hamsterrad provides the **primitive** they can all build on.

## File Map

```
hamsterrad/
├── README.md          ← This file
├── AGENTS.md          ← Agent instructions
└── docs/
    ├── index.md              ← Hub: all docs indexed here
    ├── loop-engineering-state-of-art.md  ← Research synthesis (WHY)
    ├── pi-looping-ecosystem.md            ← Existing tools + gap analysis (WHAT EXISTS)
    ├── loop-types-taxonomy.md             ← Design space (WHAT WE BUILD)
    └── case-study-scanflow.md             ← Production loops analyzed (PROOF)
```

Start at [docs/index.md](docs/index.md).

## Target Loops

| Loop | Stops When | Priority |
|-------|-----------|----------|
| **Goal** | Verifiable condition true | P0 |
| **For-each** | All items processed | P0 *(missing everywhere)* |
| **Iteration / Ralph** | DONE/BLOCKED/max iters | P1 |
| **Debug** | Test passes / error gone | P1 |
| **Research** | Answer found | P2 |
| **Dev** | Acceptance criteria met | P2 |
| **Audit / Review** | All items reviewed | P2 |
| **Refactor Sweep** | All instances migrated | P2 |

## Links

- [Documentation hub](docs/index.md)
- [Research synthesis](docs/loop-engineering-state-of-art.md)
- [Pi ecosystem gap analysis](docs/pi-looping-ecosystem.md)
- [Loop types taxonomy](docs/loop-types-taxonomy.md)
- [Case study: scanflow](docs/case-study-scanflow.md) — Real-world loop implementation analyzed
- [skaleshare](./skaleshare/) — Our doc style guide
