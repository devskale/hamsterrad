# Hamster Documentation Index

Hub for all hamster documentation. Each document covers one aspect of the looping tooling project.

## Research Phase (complete)

These documents were generated from web research (web-search skill) and X/Twitter intelligence (peep skill) in June 2026. They form the foundation we'll build on.

### 1. [Loop Engineering — State of the Art](loop-engineering-state-of-art.md)

**The big picture.** What agentic loops are, why they matter now, and who's driving the conversation.

Covers:
- Core definition: perceive → reason → act → observe → repeat
- The three-layer stack: prompt engineering → context engineering → **loop engineering**
- Five building blocks: automations, worktrees, skills, connectors, sub-agents + memory
- Named loop patterns: Ralph Wiggum loop, Goal mode (`/goal`), Stop Hook Auto-Continue, Continuous Autonomous Task Loop
- Foundational academic patterns: Ng's 4 core + Anthropic's 5 workflows + ReAct
- Augment Code's 26-pattern catalog taxonomy
- 8 agentic coding patterns with implementation examples
- Anti-patterns: God Prompt, Over-Agentification, Uncontrolled Recursion, Cognitive Surrender
- Key voices: Simon Willison, Peter Steinberger (steipete), Boris Cherny, Addy Osmani, Andrej Karpathy

**Read this first** if you need to understand why loops matter.

### 2. [Pi Looping Ecosystem — What Exists Today](pi-looping-ecosystem.md)

**The competitive landscape.** Every pi package, extension, and pattern that does anything loop-related, plus X/Twitter signal.

Covers:
- **pi-goal-x** — Full goal mode (3.6K downloads/mo), verification contracts, task lists, independent auditor
- **pi-ralph** — Hat-based multi-agent orchestration with YAML presets (feature, code-assist, debug, refactor, review)
- **pi-autoresearch** — Autonomous experiment loop inspired by karpathy/autoresearch, confidence scoring
- **rpiv-pi** — Ship-loop: research → design → plan → implement → validate (5 skills)
- **@quintinshaw/pi-dynamic-workflows** — Fan-out across 100s of subagents (103K downloads/mo)
- **X/Twitter signal** — steipete's viral "design loops" quote, ttmonk, eSaadster, cv_usk's full Loop Engineering guide, Zev_ee's "Loops" directory of 26 workflows, cost warnings ($1,400/hour horror stories), counter-signal about autonomous loops producing garbage

**Read this** to understand what we're not rebuilding and where the gaps are.

### 3. [Loop Types Taxonomy](loop-types-taxonomy.md)

**The design space.** Every loop shape we might support, with trigger/stop/shape analysis for each.

Covers:
- Goal loop — run until verifiable condition true
- For-each / list loop — batch process N items (the biggest gap in the ecosystem)
- Ralph / iteration loop — bounded retry with fresh context per iteration
- Debug loop — run until test passes or error resolved
- Research loop — deep investigation until answer found
- Dev loop — feature ticket until all acceptance criteria met
- Audit / review loop — systematic review of PR/diff set
- Refactor sweep loop — pattern-matched migration across codebase
- Scheduled / cron loop — timer-triggered recurring work

**Read this** when designing which loop primitives to build first.

## Design Phase (upcoming)

These documents will be written as we move from research to architecture.

### 4. [Case Study: Scanflow](case-study-scanflow.md)

**Real-world proof.** A production agentic workflow (kontext.one/strukt2meta) analyzed through our loop taxonomy. Three nested loops, 3-layer state model, context window management — and where hamster primitives would replace ~36% of its code.

Covers:
- Architecture: CLI → Orchestrator → Agent → Tools (3 nested loops)
- Loop 1: Tool-calling turn loop (ReAct building block)
- Loop 2: Iteration loop with retry + prune (Ralph pattern, P1)
- Loop 3: Phase-gated pipeline (Dev/Ship loop, P2)
- State management deep-dive (3 layers: working memory / artifacts / resume)
- Context window management strategy (receipt compression + emergency pruning)
- What validates from our research + what's missing
- Integration plan: near-term, medium-term, long-term
- **~36% of scanflow's code is generic loop plumbing** that hamster could replace

**Read this** to see what loops look like in production today.

### 5. [Extension Architecture](extension-architecture.md) *(not yet written)*

How we'll implement looping as pi extensions. API design, state management, event protocol, composition model, distribution format.

---

## How the Research Fits Together

```
loop-engineering-state-of-art.md  ←  "WHY: the paradigm shift"
           │
           ├──→ pi-looping-ecosystem.md  ←  "WHAT EXISTS: don't rebuild these"
           │
           ├──→ loop-types-taxonomy.md   ←  "WHAT WE'LL BUILD: the design space"
           │
           ├──→ case-study-scanflow.md   ←  "PROOF: real-world loops in production"
           │
           └──→ extension-architecture.md  ←  "HOW: implementation (TBD)"
```

## Quick Start

If you're jumping in:

1. **New to loops?** Start with [Loop Engineering State of Art §1–3](loop-engineering-state-of-art.md) — definition, stack layers, five building blocks
2. **Want to use loops now?** Read [Pi Looping Ecosystem §1–4](pi-looping-ecosystem.md) — install pi-goal-x or pi-ralph today
3. **Want a concrete example?** Read [Case Study: Scanflow](case-study-scanflow.md) — production code analyzed through our taxonomy
4. **Building here?** Read this index top-to-bottom, then [AGENTS.md](../AGENTS.md) for coding conventions
