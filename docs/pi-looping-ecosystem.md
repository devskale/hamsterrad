# Pi Looping Ecosystem — What Exists Today

Every pi package, extension, and pattern that does anything loop-related. Gap analysis for what hamster should (and shouldn't) build.

*Compiled 2026-06-10 from web research + X/Twitter intelligence.*

---

## 1. pi-goal-x — Goal Mode

| | |
|---|---|
| **Author** | @ttmonk (Thomas Monk), fork of @capyup/pi-goal |
| **Installs** | 3,635/mo · 2,319/wk |
| **Version** | 0.18.7 (June 7, 2026) |
| **Install** | `pi install npm:pi-goal-x` |

### What it does
Full goal-mode extension. `/goals`, `/sisyphus`, `/goal-set`. Auto-continues across turns until completion, pause, abort, or user interruption.

### Key features
- **Verification contracts** per goal/task — agent can't complete without evidence
- **Task lists** with recursive subtasks (configurable depth)
- **Independent completion auditor** — separate model grades the work
- **Live progress widget** — spinner, progress bar, step labels
- **Press `Esc`** to skip audit, `a` to toggle auditor per-goal
- **Disk-backed state** in `.pi/goals/`, survives reloads
- **Sisyphus mode** for ordered execution (patient, sequential)

### Commands
```
/goals <topic>          Discuss/draft a regular goal
/sisyphus <topic>       Discuss/draft a Sisyphus-style goal
/goals-set <objective>  Immediate create + start
/sisyphus-set <objective> Immediate create + start
/goal-status            Show focused goal state
/goal-list              List all open goals
/goal-focus             Choose session focus
/goal-tweak <change>    Draft revision to active goal
/goal-pause / resume / abort / clear
/goal-settings          Configure auditor model, etc.
```

### Signal
ttmonk replied to steipete's viral "design loops" post: *"I've been working on pi-goal-x for exactly this use case... using it on a daily basis for research."*

### What to learn from
- Verification contracts are the right pattern — done condition must be *evidence-based*, not trust-based
- The auditor toggle (per-goal `a` key) is smart UX — not every task needs heavy verification
- Sisyphus vs regular goal is a real distinction — ordered vs exploratory
- Task lists with subtasks add meaningful structure but complexity grows fast

### What NOT to rebuild
- Full goal lifecycle management (pause/resume/abort/focus/list) — pi-goal-x owns this
- TUI widget rendering for goals — mature, well-tested
- Disk-backed goal state in `.pi/goals/` — solved problem

### What's missing that we could provide
- A **primitive** that pi-goal-x could use instead of implementing its own auto-continue loop
- For-each iteration over task list items (pi-goal-x has tasks but no batch-for-each primitive)

---

## 2. pi-ralph — Hat-Based Multi-Agent Orchestration

| | |
|---|---|
| **Author** | @samfoy |
| **Install** | Clone to `~/.pi/agent/extensions/ralph/` |

### What it does
Ralph Wiggum pattern for Pi. Specialized "hats" (roles) hand off via events in an autonomous loop.

### Built-in presets

| Preset | Hats | Use Case |
|--------|------|----------|
| **feature** | Builder → Reviewer | General feature development |
| **code-assist** | Planner → Builder → Validator → Committer | Full TDD pipeline |
| **spec-driven** | Spec Writer → Critic → Implementer → Verifier | Spec-first dev |
| **debug** | Investigator → Tester → Fixer → Verifier | Scientific debugging |
| **refactor** | Refactorer → Verifier | Safe refactoring |
| **review** | Reviewer → Deep Analyzer | Code review |

### How it works
1. Loop starts at `starting_event` → finds matching hat
2. Hat instructions injected into system prompt
3. Agent works under that hat's guidance
4. Agent publishes event: `>>> EVENT: event_name`
5. Ralph finds next hat triggered by that event
6. Repeats until `LOOP_COMPLETE` or limits hit

### Custom presets (YAML)
```yaml
event_loop:
  starting_event: "build.start"
  completion_promise: "LOOP_COMPLETE"
  max_iterations: 50
  max_runtime_seconds: 14400

hats:
  planner:
    name: "📋 Planner"
    triggers: ["build.start"]
    publishes: ["tasks.ready"]
    instructions: |
      ## PLANNER MODE
      Your detailed instructions here...
```

### Commands
```
/ralph [preset] [prompt]   Start a loop
/ralph stop                Stop current loop
/ralph status              Show loop status
/ralph history             Iteration history
/ralph loops               Browse past loop records
/ralph presets             List available presets
/plan [idea]               PDD planning session
```

### What to learn from
- Event-driven hat handoff is a clean composition model — hats are loosely coupled
- YAML presets make loops user-configurable without code
- `completion_promise` as a magic string is simple but fragile (agent must output exact text)
- Guardrails (max iters, max runtime) are non-negotivable in any loop system
- PDD (Prompt-Driven Development) as a separate planning phase before looping is valuable

### What NOT to rebuild
- Hat-based orchestration framework — pi-ralph owns this
- YAML preset system — mature approach
- Event protocol between hats — solved

### What's missing that we could provide
- The **underlying iteration engine** that ralph's hats run on top of
- A **for-each primitive** that could power "run reviewer hat across N files"
- State persistence format that works across loop types (ralph uses its own)

---

## 3. pi-autoresearch — Autonomous Experiment Loop

| | |
|---|---|
| **Author** | @davebcn87 (inspired by karpathy/autoresearch) |
| **Install** | `pi install npm:pi-autoresearch` |

### What it does
Try idea → benchmark → keep improvements → revert regressions → repeat forever. Domain-agnostic optimization loop.

### Tools provided
| Tool | Does |
|------|-------|
| `init_experiment` | One-time config: name, metric, unit, direction |
| `run_experiment` | Runs any command, captures timing + output |
| `log_experiment` | Records result, auto-commits, updates widget |

### Session state (all in `.auto/`)
| File | Purpose |
|------|---------|
| `.auto/prompt.md` | Living doc: objective, what's been tried, dead ends |
| `.auto/measure.sh` | Benchmark script — outputs `METRIC name=number` lines |
| `.auto/log.jsonl` | Append-only log of every run (survives restarts) |
| `.auto/checks.sh` | Optional backpressure: tests, typecheck, lint after each pass |
| `.auto/hooks/` | Optional before/after iteration scripts |

### Key design decisions
- **Extension = infrastructure, Skill = domain knowledge** — one extension serves unlimited domains
- **Confidence scoring** via MAD (Median Absolute Deviation) — green ≥2.0×, yellow 1–2×, red <1.0×
- **Post-compaction recovery** — detects idle after auto-compaction, re-reads prompt.md + log tail + git log
- **Hooks are transparent to agent** — agent calls tools, hooks fire alongside

### Signal (critical counter-signal)
@robdel12: *"Look at all the PRs the Shopify CEO slopped out with pi-autoresearch. None merged."*

Autonomous loops without curation produce garbage at scale.

### What to learn from
- **Extension/Skill separation** is the right architecture — infrastructure shouldn't encode domain logic
- `.auto/` as a single self-contained session folder is elegant — survives reverts, gitignore-friendly
- **Confidence scoring** is a generalizable concept for any iterative loop (not just benchmarks)
- **Backpressure checks** (tests after each iteration) prevent optimization from breaking correctness
- **Post-compaction recovery** is essential for long-running loops — context gets summarized, agent goes idle

### What NOT to rebuild
- Experiment/benchmark-specific tooling — pi-autoresearch owns this
- Dashboard widget + fullscreen overlay — domain-specific UI
- Confidence scoring for numeric metrics — solved here

### What's missing that we could provide
- The **generic iterate-until-improved** primitive that autoresearch builds on
- The **compaction-recovery** pattern as a reusable utility any long loop can use
- The **`.auto/` state folder convention** as a shared standard

---

## 4. rpiv-pi — Ship Loop (Research → Design → Plan → Implement → Validate)

| | |
|---|---|
| **Author** | @juicesharp |
| **Install** | See rpiv-pi repo |

### What it does
5 skills forming a pipeline: research, design, plan, implement, validate. Each phase has success criteria validated independently against the working tree.

### Architecture
- **Skills** do orchestration; **subagents** compose the ship-loop
- Each phase emits pass/fail validation report — catches half-finished work
- Agent #2 can inherit context from Agent #1 via communication docs

### What to learn from
- **Phase-gated pipelines** are a distinct loop shape from goal-loops and iteration-loops
- **Independent validation per phase** catches work the implement loop misses
- **Cross-agent context inheritance** via signed docs is a pattern for multi-session continuity

### What NOT to rebuild
- The 5-phase ship pipeline itself — rpiv-pi owns this
- Phase-specific skills — domain knowledge encoded there

---

## 5. @quintinshaw/pi-dynamic-workflows

| | |
|---|---|
| **Downloads** | 103.2K/mo |
| **Install** | `pi install npm:@quintinshaw/pi-dynamic-workflows` |

### What it does
Claude Code-style dynamic workflows for Pi. Fan tasks across 100s of subagents with model routing, token/cost accounting, resume, git-worktree isolation, interactive TUI, `/deep-research`.

### What to learn from
- **Fan-out/fan-in** across subagents is a loop shape (parallel-for-each with aggregation)
- **Cost accounting per sub-agent** is essential at scale
- **Worktree isolation** is how parallel agents coexist safely

### What NOT to rebuild
- Sub-agent fan-out orchestration — this package owns it

---

## 6. Other Notable Mentions

| Package | Does | Note |
|--------|------|-------|
| [pi-subagents](https://github.com/nicobailon/pi-subagents) (103K/mo) | Sub-agent delegation with chains, parallel exec, TUI clarification | Infrastructure we may depend on |
| [@gotgenes/pi-subagents](https://github.com/gotgenes/pi-subagents) (17K/mo) | Claude Code-style autonomous sub-agents for pi | Fork with more features |
| [piolium](https://pi.dev/packages/@vigolium/piolium) (29K/mo) | Multi-phase security audits with specialist sub-agents | Domain-specific audit loop |
| [rpiv-todo](https://github.com/juicesharp/rpiv-mono) (46K/mo) | Stateful todo list with session replay, overlay | State management patterns to study |
| [roborev](https://x.com/DanKornas/status/2063584483861746066) | Post-commit review for any coding agent (incl. Pi) | External verifier pattern |
| [Loops directory](https://x.com/Zev_ee/status/206440686219650783) (@Zev_ee) | 26 ready-to-use workflow prompts (Test Until Green, Ship PR Until Green, PR Babysitter...) | Prompt-level loops, no infra — good reference for loop shapes |

---

## 7. X/Twitter Signal Summary

### The viral origin
@steipete (Peter Steinberger): *"You shouldn't be prompting coding agents anymore. You should be designing loops that prompt your agents."* — 2M views, June 2026. Almost every looping-related tweet in the last 48 hours QTs or replies to this.

### Key takes from the conversation

| Voice | Take |
|-------|------|
| **@aryaniyaps** | Built "Ultimate Pi" — plan→run→review is a loop, not a prompt. "Prompting is 2024. Orchestrating is now." |
| **@ttmonk** | Uses pi-goal-x daily for research. Direct reply to steipete. |
| **@eSaadster** | Combining native tools + /goal extension = "near autonomous machine" with 360° feedback loop |
| **@norlava** | "Build reliable long running agents w/ verification, worktrees, skills, subagents, & HIL/review gates" |
| **@cv_usk** | Most complete Loop Engineering explainer on X — all 5 components, 4-condition test, MVL approach, cost/security risks |
| **@AunySillyMe** | "Can your context travel into the agent loop, or does the agent have to rebuild you every run? 0% is configuring the human behind it." |
| **@SillyK8** (Chinese) | Predicted: harness engineering → **auto-harness engineering** — programmers write loops that make agents write their own harnesses |
| **@s6sdev** (Chinese, skeptic) | "Loop engineering is pure hype. Agent practice has always been like this." — Valid counterpoint, keeps us honest |
| **@aiSkillMarket** (Chinese) | From Prompt Engineer to Loop Engineer as the new role. Comprehensive guide. |
| **@Smartpigai** (Chinese) | Shared Boris Cherny's internal discussion on Loop Engineering — "far more valuable than normal vibe coding tutorials" |
| **@Zev_ee** | Launched **"Loops"** — directory of 26 ready-to-use agent workflow prompts |
| **@alex_prompter** | Cost warning: $1,400 in 1 hour because PM asked Cursor to tag 87 tasks. No loop detection. "The safety feature is the upsell." |
| **@santachiara1995** | Spending 1h/day labeling data for Personal Reward Model → scorer for Ralph loops + verifier for RL |
| **@novus771** (Chinese) | Published "How-To-Pi" tutorial covering agent loop, tool calling, compaction collaboration |
| **@robdel12** | **Counter-signal**: Shopify CEO's pi-autoresearch PRs — none merged. Autonomous loops without curation = garbage |
| **@vishalzone** | "The scarce skill isn't using AI, it's refusing to ship AI-generated garbage." |

### Themes from the signal

1. **Loop engineering is the new term of art** — but skepticism exists about hype
2. **Pi specifically is mentioned** as the target for multiple people's loop experiments
3. **Cost danger is real** — unbounded loops burn money shockingly fast
4. **Curation is the unsolved problem** — autonomous loops need human judgment somewhere in the cycle
5. **Context portability** is underdiscussed — the best loops carry context into each iteration automatically
6. **Auto-harness engineering** may be the next wave — loops that write the scaffolding they need

---

## 8. Gap Analysis: What Hamster Should Build

### Don't rebuild (solved elsewhere)

| Capability | Owned By | Use It |
|------------|----------|--------|
| Goal lifecycle (pause/resume/abort/focus) | pi-goal-x | Compose, don't replace |
| Hat-based multi-agent orchestration | pi-ralph | Compose |
| Experiment/benchmark loop | pi-autoresearch | Compose |
| Phase-gated ship pipeline | rpiv-pi | Compose |
| Sub-agent fan-out | pi-subagents / pi-dynamic-workflows | Depend on it |
| Post-commit review | roborev | Integrate as verifier |

### Missing everywhere (build this)

| Gap | Who Needs It | Why It Doesn't Exist |
|-----|-------------|---------------------|
| **For-each / list loop primitive** | Everyone | Nobody generalized beyond their domain |
| **Generic iteration engine** | Any loop extension | Each built their own inline |
| **Compaction-safe state survival** | Long-running loops | Only pi-autoresearch solved it, privately |
| **Maker-checker as a primitive** | Every loop type | Each implemented ad-hoc |
| **Loop cost guardrail library** | Anyone running unattended | Tool vendors incentivize overspending |
| **Standard state file convention** | Cross-loop interoperability | Each uses different format/location |
| **Loop composition API** | Advanced users | No standard way to chain loop types |

### Hamster's positioning

Hamster provides the **loop primitives** that pi-goal-x, pi-ralph, pi-autoresearch, and future loop extensions can all build on — rather than each reinventing iteration, state, guardrails, and verification independently.

Think of it as: **what express.js is to web servers, hamster is to agentic loops for pi.**
