# AGENTS.md — Hamsterrad

Agentic loop primitives for pi. See [README.md](README.md) for project overview.

## Tooling

- **Runtime:** Node.js via jiti (TS no compile step)
- **pi target:** 0.74+ (`ExtensionAPI`, `TypeBox`, `StringEnum`)
- **Deps:** `@earendil-works/pi-coding-agent`, `typebox`, `@earendil-works/pi-ai`

## Layout

```
extensions/          One loop type per file (kebab-case)
lib/                 Shared: state, guardrails, verifier
skills/              Loop-specific SKILL.md files
```

## Conventions

- **One concern per extension** — compose via `pi.events`
- **Namespace events:** `hamsterrad:done`, `hamsterrad:blocked`, `hamsterrad:iter-start`
- **State in `details` or `appendEntry`** — never module-level vars alone (breaks fork/tree)
- **Always truncate output** — 50KB / 2000 lines via `truncateHead`/`truncateTail`
- **`StringEnum`** for enums — not `Type.Union`/`Type.Literal` (Google API compat)
- **Throw errors** — don't return as content (sets `isError: true`)

## Boundaries

- **Never** run loops without `maxIterations` or `timeoutMs`
- **Never** let agent grade its own work without separate verifier
- **Never** block pipeline with `{ block: false }` — return `undefined`
- **Always** require approval for: package installs, git push, `maxIterations > 50`, file deletion

## Footguns

| Footgun | Fix |
|---------|-----|
| Module state breaks on fork | Return full state in `details`; reconstruct in `session_start` + `session_tree` |
| Stale ctx after replacement | Guard compaction/tree handlers against stale-proxy error |
| Unbounded token burn | Every loop constructor requires caps |
| Agent self-grades homework | Maker-checker split (separate model or prompt) |
| Context rot in long loops | Fresh context per iteration, git as memory |
| Missing runtime deps | All imports go in `dependencies` not `devDependencies` |

## References

- [Pi extension best practices](skaleshare/docs/pi-extensions-bestpractices.md) — Full API, anti-patterns, gotchas
- [AGENTS.md best practices](skaleshare/agentsmd_bestpractices.md) — Inclusion test, size limits
- [Loop engineering research](docs/loop-engineering-state-of-art.md)
