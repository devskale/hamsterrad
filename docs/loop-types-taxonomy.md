# Loop Types Taxonomy

Every loop shape we might support, with trigger/stop/shape analysis, implementation complexity, and priority for hamster.

*This is the design space. Not all of these will be built first. Use this to decide what to tackle when.*

---

## Decision Framework

Before building any loop type, it must pass the **4-condition test** (from cv_usk's Loop Engineering guide):

1. **Does the task repeat?** If less than weekly, a manual prompt is faster
2. **Is verification fully automated?** Can tests/linters/type-checkers reject bad output 100% of the time?
3. **Can your token budget absorb waste?** Loops produce 3-10x the tokens of single-turn work
4. **Does the agent have the right tools?** Can it check logs, run commands, inspect state?

If any answer is "no" or "maybe" → don't loop it yet. Build the manual version first.

---

## P0 — Build First

### 1. Goal Loop

Run until a verifiable condition becomes true.

```
User: /goal "All tests in test/auth pass and lint is clean"
       ↓
Agent works autonomously, turn after turn
       ↓
After each turn: separate verifier checks condition
       ↓
Condition true? → DONE ✅
Blocked?         → BLOCKED ⛔
User interrupt?  → PAUSED ⏸
Max iters?      → STOP with progress report
```

**Shape:** Open-ended, condition-gated, auto-continue across turns

**Trigger:** User sets objective + done condition (command or discussion-draft flow)

**Stops when:**
- Verifiable condition evaluates true (agent doesn't self-grade)
- Agent explicitly blocked (can't proceed, needs input)
- User interrupts (Esc / abort)
- Max iterations reached
- Token budget exhausted
- Timeout elapsed

**State model:**
```typescript
interface GoalState {
  id: string
  objective: string                    // What we're doing
  doneCondition: string                // Verifiable exit condition
  status: 'active' | 'paused' | 'done' | 'blocked' | 'aborted'
  currentIter: number                 // Which iteration we're on
  maxIterations: number               // Hard cap (default: 20)
  notes: string                       // Accumulated findings (NOTES.md)
  startedAt: ISO8601
  completedAt?: ISO8601
  verificationSummary?: string        // From verifier (evidence)
}
```

**Key design decisions:**
- **Discussion-first vs direct-set**: pi-goal-x proves both flows are needed. `/goals <topic>` discusses first; `/goals-set <obj>` starts immediately.
- **Verifier model**: Should be configurable — same model (cheap), different model (strong), or shell script (deterministic). Shell script verifier is most trustworthy.
- **Auto-continue mechanism**: After each `turn_end`, if goal is active and not done, inject continuation instructions as a follow-up user message. This is how pi-goal-x does it.
- **Context rot mitigation**: Each iteration should re-read the goal state file + recent git log, not rely on accumulated conversation context alone.

**What pi-goal-x teaches us:**
- Verification contracts per goal are powerful — encode them in the done condition
- Task lists inside goals add structure but complexity grows fast (subtasks, recursive depth)
- The auditor toggle (per-goal) is essential — not every task needs heavy verification
- Sisyphus mode (ordered execution) is a variant, not a separate loop type

**Complexity:** ★★★☆☆ (state management + verifier + auto-continue + lifecycle)

---

### 2. For-Each / List Loop

Process N items from a list, running the same operation on each.

```
User: /foreach "Run add-error-handling on each file in src/handlers/"
       ↓
Agent picks next unprocessed item from list
       ↓
Executes the operation on that item
       ↓
Records result (pass/fail/skip) 
       ↓
Next item... until all processed or stop condition
```

**Shape:** Bounded batch, item-at-a-time, accumulates results

**Trigger:** User provides list (file paths, issue numbers, function names, etc.) + operation prompt

**Stops when:**
- All items processed → DONE
- Error threshold exceeded (e.g., >30% failures) → BLOCKED
- User interrupt
- Max iterations (default: list length × 2)

**State model:**
```typescript
interface ForEachState {
  id: string
  items: ForEachItem[]                // The full list
  currentItemIndex: number            // -1 = not started
  results: ItemResult[]              // Accumulated outcomes
  operation: string                  // What to do on each item
  stopCondition?: string             // e.g., "stop after 3 consecutive failures"
  status: 'idle' | 'running' | 'done' | 'stopped' | 'error'
}

interface ForEachItem {
  id: string                          // e.g., file path
  label: string                       // Display name
  status: 'pending' | 'running' | 'done' | 'failed' | 'skipped'
  result?: string                    // Output from processing
  error?: string                     // Error if failed
  iterStarted?: ISO8601
  iterCompleted?: ISO8601
}
```

**Why this is THE gap:**
- Every existing pi looping tool operates on ONE thing at a time
- Real workflows need: "audit these 15 files", "add tests to these 8 modules", "migrate these 20 components"
- Nobody has generalized this for pi
- It's the simplest loop to understand AND the most universally useful

**Key design decisions:**
- **List source**: Inline (user pastes items), glob (`src/**/*.ts`), git (`changed files since main`), command output (`grep -rl TODO src/`)
- **Ordering**: Sequential (safe, predictable) vs parallel via sub-agents (fast, collision risk). Start sequential.
- **Failure mode**: Stop-on-first-error (strict) vs continue-and-report (lenient). Both needed.
- **Resume**: If interrupted at item 7 of 20, restart at item 7. State file makes this trivial.
- **Result aggregation**: Summary table at completion: N passed, M failed, K skipped, with details per item.

**Variants:**

| Variant | Example | Difference |
|---------|---------|------------|
| **File-for-each** | Add error handling to N files | Items = file paths, op = edit |
| **Test-for-each** | Run tests for N modules | Items = test paths, op = run + check |
| **PR-for-each** | Review N PRs | Items = PR numbers, op = review |
| **Pattern-for-each** | Replace pattern across codebase | Items = grep matches, op = refactor |

**Complexity:** ★★☆☆☆ (list + iterate + accumulate + resume)

**This should be our FIRST build.** Smallest scope, highest utility, biggest gap in ecosystem.

---

## P1 — Build Second

### 3. Iteration / Ralph Loop

Bounded retry with fresh context per iteration. The classic Ralph Wiggum pattern.

```
Script loop:
  for i in 1..MAX_ITERS:
    agent reads TASK.md + NOTES.md
    agent does one smallest useful step
    agent appends findings to NOTES.md
    if agent creates DONE file → exit 0
    if agent creates BLOCKED file → exit 2
    git commit --all -m "Iteration $i"
  done
  echo "Hit MAX_ITERS"
```

**Shape:** Bounded retry, fresh session each iteration, git-as-memory

**Trigger:** User provides task description + optional constraints + done conditions

**Stops when:**
- Agent creates `DONE` file with summary
- Agent creates `BLOCKED` file with blocker details
- `MAX_ITERS` reached (default: 20)
- Timeout elapsed

**State model:**
```typescript
interface RalphState {
  taskFile: string                   // TASK.md path
  notesFile: string                  // NOTES.md path
  stateDir: string                   // Working directory (./ralph/)
  runLog: string                     // RUN_LOG.md path
  currentIter: number
  maxIterations: number
  status: 'running' | 'done' | 'blocked' | 'exhausted'
  doneAt?: ISO8601
  summary?: string                   // From DONE file
  blocker?: string                   // From BLOCKED file
}
```

**Key design decisions:**
- **Outer script vs inner extension**: ralph-loop can be a bash script that invokes pi (like Codex version) OR a pi extension that manages the loop internally. The extension approach gives us TUI integration, widget rendering, and event protocol participation.
- **One step per iteration**: The "smallest useful step" discipline prevents the agent from trying to do too much and failing silently.
- **Git commit between iterations**: Each iteration is a checkpoint. `git diff` shows progress. Any iteration can be reverted.
- **Fresh context = clean reasoning**: Each iteration starts without the baggage of previous failed attempts (except what's written in NOTES.md).

**What pi-ralph teaches us:**
- Hat-based orchestration is a superset — a ralph loop is a single-hat (or no-hat) iteration loop
- YAML presets are how users customize behavior
- Event protocol (`>>> EVENT:`) is one way to hand off; file signaling (`DONE`/`BLOCKED`) is simpler and more universal

**When to use vs goal loop:**
- **Goal loop** = open-ended, condition-driven, same session context, "keep going until tests pass"
- **Ralph loop** = bounded, task-driven, fresh context each time, "make progress on this defined task, I'll check back"

**Complexity:** ★★★☆☆ (iteration management + file signaling + git integration)

---

### 4. Debug Loop

Run until a specific failure disappears.

```
User: /debug "test/auth/login.spec.ts fails intermittently"
       ↓
Agent investigates: reads test, reproduces, hypothesizes
       ↓
Agent implements fix
       ↓
Agent runs test
       ↓
Passes? → DONE ✅
Still fails? → Back to investigate (new hypothesis)
       ↓
Max attempts or blocker → STOP
```

**Shape:** Hypothesis-driven iteration, shrinking problem space

**Trigger:** Failing test, error message, bug report, or unexpected behavior

**Stops when:**
- Test passes (or error disappears) for N consecutive runs (default: 3)
- Root cause identified but outside scope
- Max iterations (default: 10)
- Agent determines it's a flake / environment issue, not a code bug

**State model:**
```typescript
interface DebugState {
  target: string                    // Test name, error message, or reproduction cmd
  hypothesisHistory: Hypothesis[]   // Tried approaches and outcomes
  consecutivePasses: number          // How many times the fix held
  requiredPasses: number            // Default: 3
  originalError: string             // Snapshot of initial failure
  status: 'investigating' | 'fixing' | 'verifying' | 'done' | 'blocked'
}
```

**Key design decisions:**
- **Hypothesis tracking** is what separates a debug loop from a generic goal loop. Recording what you tried and why it failed prevents circular reasoning (the #1 plague of agentic engineering).
- **Consecutive passes requirement** prevents declaring victory on a flaky test. 3 is default, configurable.
- **Reproduction command** must be explicit and deterministic. "Run the test" is better than "check if it works."
- **Scientific method structure**: Observe → Hypothesize → Experiment → Conclude. The loop enforces this rhythm.

**pi-ralph's debug preset** covers this well. Our version should be more opinionated about hypothesis tracking.

**Complexity:** ★★★☆☆ (hypothesis tracking + verification cadence + reproducibility)

---

## P2 — Build When Foundation Is Solid

### 5. Research Loop

Deep investigation until an answer is found or sources exhausted.

```
User: /research "Compare WebSocket libraries for Node.js in 2026"
       ↓
Agent searches, reads, takes notes
       ↓
Synthesizes findings into structured notes
       ↓
Gaps remain? → Search more, read deeper
       ↓
Answer found or sources exhausted → DONE
```

**Shape:** Expanding information gathering, synthesis-driven stopping

**Trigger:** Research question, unknown technology decision, competitive analysis

**Stops when:**
- Answer meets confidence threshold (configurable, default: "high")
- No new information found in last N iterations (diminishing returns)
- All reasonable sources exhausted
- Max iterations or token budget

**State model:**
```typescript
interface ResearchState {
  question: string
  findings: Finding[]               // Structured discoveries
  sourcesConsulted: Source[]        // URLs, papers, docs read
  gaps: string[]                    // Unanswered sub-questions
  confidence: 'low' | 'medium' | 'high'
  maxIterations: number
  status: 'gathering' | 'synthesizing' | 'done' | 'exhausted'
}
```

**Key design decisions:**
- **Structured findings** beat freeform notes. Template: claim, evidence, source, confidence.
- **Diminishing returns detection**: If iteration N+1 produces no new claims that weren't already in iteration N's findings, stop.
- **Source deduplication**: Track what URLs/papers have been read. Don't re-read.
- **Confidence scoring**: Simple heuristic — number of independent sources agreeing × recency weight.

**Existing coverage:** rpiv-pi has a research skill. pi-web-access provides search/fetch tools. This loop ties them together.

**Complexity:** ★★★☆☆ (source tracking + synthesis + diminishing-return detection)

---

### 6. Dev Loop (Feature Ticket → Ship)

Full feature lifecycle: ticket → spec → implement → verify → ship.

```
User: /dev "Add structured logging to auth module"
       ↓
Agent reads ticket / writes spec (if needed)
       ↓
Implements (possibly across multiple files)
       ↓
Runs tests, lint, type-check
       ↓
All green? → Commit, optionally PR
       ↓
Failures? → Fix (inner debug loop)
       ↓
All acceptance criteria met → DONE
```

**Shape:** Multi-phase pipeline with inner loops, acceptance-criteria-gated

**Trigger:** Feature ticket, issue, or user story with clear acceptance criteria

**Stops when:**
- All acceptance criteria verified
- Blocked (dependency missing, requirements unclear)
- Max iterations (feature work is expensive — default low: 10)

**State model:**
```typescript
interface DevLoopState {
  ticketRef: string                  // Issue URL or ticket ID
  acceptanceCriteria: string[]       // Must all pass
  phase: 'discovery' | 'planning' | 'implementing' | 'verifying' | 'shipping'
  criteriaStatus: Record<string, boolean>  // criterion → passed
  commits: string[]                  // Git commits made
  prUrl?: string                    // If opened
  status: 'active' | 'done' | 'blocked' | 'aborted'
}
```

**Key design decisions:**
- **Acceptance criteria must be explicit before coding begins.** If the ticket doesn't have them, the loop's first job is to write them and get confirmation.
- **Phase transitions are gates.** You don't implement until criteria are defined. You don't ship until all criteria pass.
- **Inner debug loop**: When a test fails during verification, delegate to the debug loop rather than inventing ad-hoc fixing logic.
- **PR creation is optional** — some orgs want commits directly to branch, others want PRs.

**Existing coverage:** rpiv-pi's ship-loop (5 skills) covers this comprehensively. Hamster's dev loop would be a lighter, composable alternative using our primitives.

**Complexity:** ★★★★☆ (multi-phase + criteria tracking + inner loop delegation)

---

### 7. Audit / Review Loop

Systematic review of a diff set against multiple dimensions.

```
User: /audit "Review changes in src/auth/"
       ↓
Agent reads diff (git diff, PR, or file range)
       ↓
Reviews against each dimension:
  - Correctness (bugs?)
  - Security (injections, auth bypass?)
  - Performance (N+1, missing indexes?)
  - Style (conventions?)
  - Test coverage (regressions?)
       ↓
Produces structured review report
       ↓
All dimensions reviewed → DONE
```

**Shape:** Multi-dimensional systematic examination, checklist-driven

**Trigger:** PR, diff range, set of files, or branch comparison

**Stops when:**
- All configured review dimensions examined
- Max files reviewed (for large diffs)

**State model:**
```typescript
interface AuditState {
  target: string                    // Diff ref: PR#, branch, file glob
  dimensions: AuditDimension[]       // What to check
  findings: Finding[]                // Issues discovered
  reviewedFiles: string[]           // Files examined so far
  status: 'scanning' | 'reviewing' | 'done'
}

interface AuditDimension {
  name: string                      // e.g., "Security"
  reviewerPrompt: string             // Instructions for this dimension
  severity: 'critical' | 'warning' | 'info' | 'style'
  reviewed: boolean
}
```

**Key design decisions:**
- **Dimensions are pluggable** — user configures which review angles to apply. Built-in defaults: security, performance, correctness, style, test coverage.
- **Severity classification** — not all findings are equal. Critical blocks merge. Info is advisory.
- **Maker-checker split is mandatory here** — the author's agent must NOT be the auditor. Different model, different prompts, or at minimum different system instructions.
- **Integration with roborev-style post-commit review** — audit loop can be the interactive version; roborev handles the automated background version.

**Existing coverage:** pi-ralph's `review` preset, piolium (security audit), rpiv-advisor (second opinion).

**Complexity:** ★★★☆☆ (multi-dimensional + severity classification + maker-checker)

---

### 8. Refactor Sweep Loop

Find all instances of a pattern and migrate them systematically.

```
User: /sweep "Replace datetime.now() with timezone-aware utcnow() in Python files"
       ↓
Agent finds all instances (grep/AST scan)
       ↓
For each instance:
  - Apply transformation
  - Run related tests
  - Pass? → Keep. Fail? → Revert this instance.
       ↓
All instances migrated (or skipped with reason) → DONE
       ↓
Create summary PR with all changes
```

**Shape:** Pattern-match + transform + verify-per-instance, batched

**Trigger:** Pattern to find + replacement rule (or "figure out the replacement")

**Stops when:**
- All instances processed (migrated, skipped, or confirmed irrelevant)
- Error rate exceeds threshold (transformation is wrong)
- Max instances (safety cap for large codebases)

**State model:**
```typescript
interface SweepState {
  pattern: string                    // What to find (regex, AST pattern, or description)
  transformation: string              // What to replace with
  scope: string                      // File glob or directory
  totalInstances: number
  processed: SweepResult[]           // One per instance
  stats: { kept: number; reverted: number; skipped: number }
  status: 'scanning' | 'transforming' | 'verifying' | 'done' | 'rolled-back'
}
```

**Key design decisions:**
- **Per-instance verification** — run the relevant tests after EACH change, not just at the end. One broken migration shouldn't invalidate the whole sweep.
- **Automatic rollback** — if tests fail after a transformation, revert that specific instance and mark it skipped. Continue with remaining instances.
- **Summary PR** — group all kept changes into one atomic PR. Each commit in the PR maps to one instance.
- **Dry-run mode** — first pass just finds and reports instances without changing anything. Essential for sweeps that might touch hundreds of files.

**Existing coverage:** pi-ralph's `refactor` preset. This is the most natural fit for a for-each loop with refactor-specific semantics.

**Complexity:** ★★★☆☆ (pattern matching + per-instance verify + rollback + batch PR)

---

## P3 — Future / Integration Work

### 9. Scheduled / Cron Loop

Timer-triggered recurring work.

```
User: /loop-schedule "Triage CI failures every weekday at 9am"
       ↓
System triggers at cron interval
       → Agent triages, writes findings to TODO.md
       → Next trigger...
```

**Shape:** Time-gated, recurring, fire-and-forget (with persistent state)

**Trigger:** Schedule definition (cron expression or interval)

**Stops when:** Never (until explicitly cancelled) — or after max cumulative iterations

**Notes:** Partially exists in pi-goal-x (auto-continue across turns, not time-based). Requires either:
- System-level cron that invokes pi non-interactively (`pi --mode json`)
- Or an in-process timer (keeps pi running)

This is P3 because it requires headless/non-TUI mode to be reliable, which adds significant complexity. Start with manually-triggered loops; add scheduling once the foundation is solid.

**Complexity:** ★★★★☆ (scheduling + headless mode + persistent process)

---

## Composition: How Loops Combine

The real power isn't individual loop types — it's composition:

### Example: Autonomous Feature Development

```
/dev loop ("Add user auth")
  ├── [discovery phase] → /research loop (what libs do we use?)
  ├── [planning phase] → /foreach loop (review similar patterns in codebase)
  ├── [implementing phase] → /ralph loop (iterate on implementation)
  │   └── [test fails] → /debug loop (fix the failure)
  ├── [verifying phase] → /foreach loop (run test suite per module)
  └── [shipping phase] → /audit loop (security + correctness review)
```

### Example: Codebase Health Sweep

```
/scheduled (weekly)
  ├── /sweep loop ("replace deprecated API X")
  ├── /foreach loop (run linter on every file, report violations)
  └── /audit loop ("review all changes from this week")
```

### Composition principles

1. **Loops can contain loops** — outer loop manages phases, inner loops handle granular work
2. **State files compose** — each loop writes to its own dir; parent loop reads child summaries
3. **Stop conditions bubble up** — inner BLOCKED → parent decides whether to skip, retry, or escalate
4. **Token budgets are hierarchical** — parent allocates budget to children; child reports consumption back

---

## Implementation Order Recommendation

Based on gap analysis, complexity, and value:

| Order | Loop | Why First |
|-------|------|-----------|
| **1** | **For-Each** | Biggest gap, simplest, highest immediate utility |
| **2** | **Goal Loop** | Foundational — other loops use goals as stopping conditions |
| **3** | **Debug Loop** | Common use case, builds on goal loop primitives |
| **4** | **Ralph / Iteration Loop** | Builds on for-each + goal, adds fresh-context pattern |
| **5** | **Audit Loop** | Uses maker-checker primitive, valuable standalone |
| **6** | **Research Loop** | Useful but less urgent than dev-focused loops |
| **7** | **Dev Loop** | Composes goals + debug + foreach — needs foundations first |
| **8** | **Refactor Sweep** | Specialized for-each with rollback — build after for-each |
| **9** | **Scheduled Loop** | Needs headless reliability — last |

The first two (for-each + goal) form the primitive foundation. Everything else builds on top.
