# Planning Like Coding

> Treat agent orchestration like writing a program. The main agent is `main()`. Sub-agents are function calls. Context isolation = scope.

A Claude Code skill that turns multi-dimensional analysis tasks into structured, parallel sub-agent pipelines. Instead of one agent slogging through 5 directories sequentially and drowning in search traces, you decompose the task like a program — each independent dimension becomes a function, each function runs as an isolated sub-agent, and the main agent only sees the return values.

## The Problem

When you ask an agent to "audit X, Y, and Z across the codebase," the agent reads every file, runs every search, and holds all intermediate results in its context window. This:
- **Burns context**: Search traces and half-read files clog the window
- **Slows down**: Sequential execution of independent tasks
- **Degrades quality**: Context saturation leads to missed details

## The Solution

**Write pseudocode first, then dispatch each function as a sub-agent.**

| Code | Agent |
|------|-------|
| Function signature | Pseudocode: goal + input + return type |
| Arguments | Task description + expected output format passed to sub-agent |
| Function body | Sub-agent's internal search / read / reasoning |
| Local variables | Sub-agent's context window (invisible to parent) |
| Return value | Structured JSON matching the format main agent defined |
| `main()` assembly | Main agent combines return values into final output |

Sub-agents return **only JSON** — no prose, no Markdown tables. The main agent assembles the final report from clean structured data, never seeing the search traces.

## Install

Copy `SKILL.md` into your Claude Code skills directory:

```
mkdir -p ~/.claude/skills/planning-like-coding/examples
cp SKILL.md ~/.claude/skills/planning-like-coding/
cp examples/vault-health-check.md ~/.claude/skills/planning-like-coding/examples/
```

Restart Claude Code or use `/skills` to verify it's loaded.

## Example

See [`examples/vault-health-check.md`](examples/vault-health-check.md) for a complete walkthrough: pseudocode → 5 parallel sub-agents → assembly.

Stats from that run:

| Metric | Value |
|--------|-------|
| Sub-agents | 5 |
| First-try success | 5/5 |
| Context isolation ratio | 59:1 |
| Main agent tokens | ~1,200 |
| Sub-agent tokens (invisible to main) | ~71,000 |
| Time saved vs. sequential | ~4x |

**59:1** means for every token the main agent saw, 59 tokens of search traces stayed inside sub-agent contexts.

## When to Use

- 3+ independent exploration dimensions
- Scanning / counting across non-overlapping directories or files
- Collecting data from multiple sources with a single assembly step
- Audits, health checks, investigations, cross-cutting comparisons

## When NOT to Use

- Single-step tasks
- Steps are strongly dependent AND each can be done with a single command
- User gave precise step-by-step instructions
- Main agent would need to fetch data solely to pass to sub-agents (cost shift, not savings)

## Structure

```
planning-like-coding/
├── SKILL.md                          # Skill definition + methodology
└── examples/
    └── vault-health-check.md         # Complete end-to-end walkthrough
```

## License

MIT
