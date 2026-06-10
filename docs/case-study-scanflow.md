# Case Study: Scanflow — A Real-World Loop Implementation

Scanflow is a production agentic workflow system at `kontext.one/python-utils/packages/strukt2meta/scanflow/`. It processes German public tender documents (Ausschreibungs-Analyse-Bogen / AAB) through multi-step LLM-powered analysis.

**This is a living case study.** As we build hamsterrad primitives, we'll update this with integration plans.

*Analyzed 2026-06-10.*

---

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│  CLI (__main__.py)                          │
│  uv run scanflow lampen --flow scan -v      │
│  --step aab | --flow scan | --resume        │
└───────────┬─────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────┐
│  Orchestrator (orchestrator.py)             │
│  run_workflow(steps[]) — phase-gated pipe   │
│  run_single_step(step) — single step mode   │
│  _evaluate_step() — state-key verification  │
└───────────┬─────────────────────────────────┘
            │ per step: agent.reset() + agent.loop()
            ▼
┌─────────────────────────────────────────────┐
│  Agent (agent.py)                           │
│  .loop() — retry + prune wrapper            │
│  .run()  — tool-calling turn loop           │
│  _build_llm_messages() — context window mgmt│
│  _prune_conversation() — receipt compression│
└───────────┬─────────────────────────────────┘
            │ per turn: LLM call → tool exec → save
            ▼
┌─────────────────────────────────────────────┐
│  Tools (tools/)                             │
│  list, search, read, read_lines             │
│  save_state, read_state, finish             │
│  docs.py, context.py, state_tools.py         │
└─────────────────────────────────────────────┘

State layers:
  AgentState (in-memory dict) → .scanflow.state.json
  PersistedState (named results) → scanflow.json
  Resume state (full conversation) → .scanflow.resume.json
```

---

## The Three Nested Loops

### Loop 1: Tool-Calling Turn Loop (`Agent.run()`)

**Type:** Basic ReAct / Tool-Use loop (building block)

```python
for self.turn in range(self.turn + 1, max_turns + 1):
    assistant_msg = self._call_llm(tools=step_schemas)     # reason
    for tool_call in assistant_msg.tool_calls:
        result = self._execute_tool(tool_call, ...)          # act
        # observe: result appended to messages
        if fn_name == "finish" and self.ctx.finished:       # done?
            return self.result
    self.save_state(state_path)                              # persist
```

**Characteristics:**
- Bounded by `max_turns` (default 30 per `run()`, 20 from CLI)
- Serial tool execution (one tool per LLM response)
- State saved after **every tool call** — Ctrl+C safe, resume works
- Done condition: agent calls `finish` tool with result data
- No explicit blocked condition — just runs until finish or max_turns

**Maps to:** Foundational ReAct pattern. Every agentic system has this. Not a hamsterrad primitive itself — it's the substrate.

### Loop 2: Iteration Loop with Retry (`Agent.loop()`)

**Type:** Ralph / Iteration loop (our taxonomy P1)

```python
retries = 0
while retries < max_retries:              # outer retry bound
    result = self.run(step, ...)           # inner turn loop
    
    if result is not None:                 # finished!
        return result
    
    total_chars = sum(len(str(m.content)) for m in self.messages)
    if total_chars > prune_chars:          # context rot defense
        self._prune_conversation(keep_last=10)
    
    retries += 1
    time.sleep(2 ** retries)               # exponential backoff
    self.messages.append(ChatMessage(       # nudge to continue
        role="user",
        content="Continue. Call `finish` with your results so far.",
    ))
```

**Characteristics:**
- Bounded by `max_retries` (default 3) × `max_turns` (20) = max ~60 LLM calls
- **Context rot mitigation via pruning** — old tool results compressed to receipts ("Read filename lines 1-50 (4200 chars)")
- **Exponential backoff** on retry (2s, 4s, 8s)
- **Resume-safe** — loads full conversation from `.scanflow.resume.json`
- Nudge message injects "continue with what you have" pressure on final retry

**Maps to:** Our **Ralph/Iteration loop (P1)**. This IS a ralph loop — bounded retry, state-on-disk, fresh-context-via-prune (instead of ralph's fresh-context-via-restart).

**Key difference from canonical Ralph:** Canonical Ralph restarts the **entire process** (new LLM session) each iteration. Scanflow prunes the **conversation in-process** and continues. Trade-off:
- Scanflow approach: faster (no cold start), but residual context can bias
- Ralph approach: cleaner reasoning, but loses all conversational context each time

Both are valid. Hamsterrad should support **both modes** as configuration on the iteration loop.

### Loop 3: Phase-Gated Pipeline (`Orchestrator.run_workflow()`)

**Type:** Dev loop / Ship loop (our taxonomy P2)

```python
for i, step in enumerate(steps):
    self.agent.reset()                     # FRESH CONTEXT per step
    result = self.agent.loop(step, ...)     # run iteration loop
    status = self._evaluate_step(step, state, result is not None)
    
    if status == "failed":                 # hard gate
        return None                        # abort pipeline
    # partial → continue (lenient)
    # done     → proceed to next step
```

**Characteristics:**
- **Ordered phases** — steps run sequentially, no parallelism
- **Fresh context per phase** — `agent.reset()` clears messages between steps
- **State carries across phases** — `AgentState` (in-memory dict) persists; `PersistedState` writes to `scanflow.json`
- **Phase evaluation** — `_evaluate_step()` checks if expected state keys were set
- **Hard fail on broken step** — aborts entire workflow (configurable: could be lenient)
- **Resume at step level** — if interrupted mid-workflow, re-run current step from saved state

**Maps to:** Our **Dev loop (P2)** and **rpiv-pi's ship-loop**. Phase-gated pipeline with independent verification per phase.

**Built-in workflow ("scan"):**
```
Step 1: finde_aab  → Find AAB documents, produce aab_files list
Step 2: extract_meta → Extract metadata from found AAB files
```

Two-phase pipeline. Each phase has its own Step definition with allowed tools.

---

## State Management Deep-Dive

Scanflow has **three state layers**, which is instructive for hamsterrad's state design:

### Layer 1: AgentState (in-memory working memory)

```python
class AgentState:
    """Key-value store. Saved to .scanflow.state.json."""
    def set(key, value)    # e.g., state.set("aab_files", [...])
    def get(key)           # e.g., state.get("meta")
    def to_dict()          # serialize for persistence
```

- Used by tools via `ctx.state.set()` / `ctx.state.get()`
- Carries across orchestrator steps (survives `agent.reset()`)
- Written to `.scanflow.state.json` on every `save_state()`
- Simple key-value dict — no schema, no versioning

### Layer 2: PersistedState (named results accumulator)

```python
class PersistedState:
    """Named keys merged into scanflow.json."""
    def write(name, data)   # e.g., ps.write("aab_files", [...])
    def read(name)          # e.g., ps.read("meta")
```

- Output format — what the user actually cares about
- Append-only merge semantics (each `write` adds/updates one key)
- Survives across multiple workflow runs
- This is the "artifact" layer vs AgentState being the "working memory" layer

### Layer 3: Resume State (full conversation snapshot)

```python
# .scanflow.resume.json
{
    "meta": {
        "turn": 12,
        "project_dir": "/path/to/lampen",
        "state": { "aab_files": [...], ... }
    },
    "messages": [ /* full serialized ChatMessage[] */ ]
}
```

- Full conversation history for exact resume point recovery
- Includes both metadata (turn number, project dir) and all messages
- Deleted on successful completion (transient)
- Loaded by `--resume` flag or automatically when file exists

### What hamsterrad can learn from this

| Insight | Application to Hamsterrad |
|---------|----------------------|
| **3-layer state is right size** | Working memory (per-iteration) + artifacts (cross-iteration) + resume (full recovery) |
| **JSON files are sufficient** | No need for SQLite or other DB — JSON is debuggable, git-friendly, human-readable |
| **Transient resume state** | Delete on success, keep on failure — same pattern we should use |
| **State survives reset** | `agent.reset()` clears conversation but NOT agent state — important distinction |

---

## Context Window Management

Scanflow's `_build_llm_messages()` is worth studying in detail — it solves context rot without restarting:

```
Messages sent to LLM:
  [0] System prompt                          (always present)
  [1] State summary + turn counter           (rebuilt each call)
  [2] Original step goal                     (always present)
  [3] Last N turns in full fidelity          (RECENT_TURNS = 6)
  [4] Older turns: tool results → receipts   (compressed to 1 line each)
  [5] Text-only assistant messages: dropped  (noise removal)
```

Plus `_prune_conversation()` as emergency overflow:

```python
# When total > 80K chars:
# Keep: system, user goal, "[earlier summarized]" placeholder, last 10 messages
# Drop: everything else
```

**This is a sophisticated approach.** For hamsterrad's iteration loop, we should offer both strategies:
1. **Prune-in-place** (scanflow style) — keeps warm context, compresses old turns
2. **Fresh-restart** (canonical Ralph style) — clean slate each iteration, uses NOTES.md/git for continuity

---

## Step System

Each step is a `(goal, tools)` pair:

```python
class Step:
    def __init__(self, goal: str, tools: list[str]):
        self.goal = goal       # Natural language objective
        self.tools = tools     # Allowed tool names for this step
```

Steps are registered in `steps/__init__.py`:

```python
STEPS = {
    "aab": FINDE_AAB_STEP,           # "Finde alle AAB-Dokumente..."
    "extract_meta": EXTRACT_META_STEP, # "Extrahiere Metadaten aus den gefundenen AAB-Dateien..."
}
```

Workflows compose steps:

```python
# workflows/scan.py
STEPS = [FINDE_AAB_STEP, EXTRACT_META_STEP]
```

**Missing from current step system:**
- No declarative `expects` field — evaluation uses string matching on goal text
- No `depends_on` field — steps are always sequential
- No `parallel` flag — no fan-out capability
- No timeout per step — only global `max_turns`

These are all things hamsterrad's step/loop primitives could standardize.

---

## What Scanflow Validates From Our Research

| Research Claim | Scanflow Evidence |
|---------------|-------------------|
| *"Every loop needs max iterations"* | ✓ `max_turns=20`, `max_retries=3` |
| *"State must be on disk"* | ✓ 3 JSON state files, saved every turn |
| *"Fresh context prevents rot"* | ✓ `agent.reset()` between steps + pruning |
| *"Resume from interrupt is non-negotiable"* | ✓ `--resume` + `.scanflow.resume.json` |
| *"Git as memory" would help* | ✗ No git integration — state is JSON-only |
| *"Maker ≠ checker"* | ⚠️ Partial — `_evaluate_step()` checks state keys, but same agent produced them |
| *"Loops compose"* | ✓ 3 nested loops (tool-calling → iteration → pipeline) |
| *"For-each is the biggest gap"* | ✓ Can only process one item/step at a time |

---

## Integration Opportunities

### Near-term (hamsters first primitives)

| Hamsterrad Primitive | How Scanflow Would Use It |
|-------------------|--------------------------|
| **foreach-loop** | Process all AAB files in batch instead of sequentially discovering then extracting |
| **guardrails.ts** | Token/cost budget cap per workflow run (currently unbounded) |
| **Standardized state format** | Replace ad-hoc `.scanflow.*.json` with hamsterrad state convention |
| **Done/Blocked events** | Replace string-matched `_expected_key_for()` with declarative `expects` |

### Medium-term (composition)

| Capability | Benefit |
|-----------|---------|
| **Debug loop as inner loop** | When extraction fails on one AAB, auto-debug instead of failing the whole step |
| **Audit loop post-run** | Automatic quality check of extracted metadata before writing to scanflow.json |
| **Maker-checker split** | Use cheaper model for extraction, stronger model for verification pass |

### Long-term (if scanflow adopts hamsterrad)

| Change | Why |
|--------|-----|
| Replace `Agent.loop()` with hamsterrad iteration engine | Eliminate ~150 lines of retry/prune/resume plumbing |
| Replace `Orchestrator` with hamsterrad dev-loop primitive | Get phase gating, parallel steps, dependency graph for free |
| Add foreach for bulk operations | "Extract metadata from all 200 AAB files in this directory" becomes one command |
| Standard events protocol | `hamsterrad:done`, `hamsterrad:blocked`, `hamsterrad:iter-start` enable TUI widgets, monitoring, external hooks |

---

## Code Statistics

| File | Lines | Role |
|------|-------|------|
| `agent.py` | ~280 | Core: 3 nested loops, context management, state persistence |
| `orchestrator.py` | ~110 | Phase-gated pipeline with step evaluation |
| `state.py` | ~80 | 3-layer state system (AgentState + PersistedState + file I/O) |
| `__main__.py` | ~140 | CLI with --step/--flow/--resume/--fresh/--show |
| `workflows/scan.py` | ~15 | 2-step workflow definition |
| `steps/*.py` (5 files) | ~200 total | Step definitions + individual logic |
| **Total core** | **~825** | Excluding tools (which are domain-specific) |

**Of these ~825 lines, roughly 300 are loop/retry/state/plumbing infrastructure** that hamsterrad primitives could replace. That's ~36% of the codebase that's generic loop machinery, not domain logic.

---

## Key Takeaways for Hamsterrad Design

1. **Scanflow proves the patterns work** — it's running in production processing real tender documents. Loops aren't theoretical.
2. **36% of its code is generic loop plumbing** — this is the exact problem hamsterrad solves. If those 300 lines became 30 lines of hamsterrad imports, scanflow gets smaller AND more capable.
3. **The 3-layer state model is the right abstraction** — working memory, artifacts, resume. Hamsterrad should formalize this.
4. **Prune-in-place vs fresh-restart are both needed** — don't pick one. Offer both as iteration loop config.
5. **Step evaluation is underspecified** — string-matching goal text for expected outputs is fragile. Declarative `expects` on steps is better.
6. **No cost awareness is dangerous** — a long-running scanflow on an expensive model could burn significant budget with no cap. Guardrails are the first thing any loop system needs.
