# Planning Like Coding

Treat agent orchestration like writing a program. Write pseudocode to decompose tasks, dispatch each function as an isolated sub-agent, assemble structured results — without polluting the main context.

## The Problem

When you ask an agent to "audit X across the vault, check Y, and scan Z," the agent reads every file, runs every search, and holds all intermediate traces in context. This burns tokens, runs serially, and degrades quality as context saturates.

## The Solution

```
main():
    parallel:
        result_A = func_A(input) → {field1: [items], field2: int}
        result_B = func_B(input) → [{field3: type, field4: type}]
    serial:
        result_C = func_C(result_A.field1) → {field5: type}
    return assemble(result_A, result_B, result_C)
```

| Code | Agent |
|------|-------|
| Function signature | Pseudocode: goal + input + return type |
| Function body | Sub-agent's internal search (invisible to parent) |
| Return value | Structured JSON: `{meta: {evidence_count, source_count, contradiction_count}, data: {...}}` |
| `main()` assembly | Main agent merges JSON returns into final output |

Sub-agents return only JSON. Main agent never sees search traces, file reads, or intermediate reasoning. Context isolation ratio: **~59:1**.

## Trace File

Every invocation writes `<project>/.claude/traces/YYYY-MM-DD-HHmmss-<slug>.md` — a human-readable log of pseudocode, per-function execution status, findings, and aggregate stats. The model never reads this file; it's for user audit only.

## Install

```bash
git clone https://github.com/m1908293028-boop/planning-like-coding.git \
  ~/.claude/skills/planning-like-coding
```

Or with skill managers:

```bash
npx skills add m1908293028-boop/planning-like-coding
# or
skilluse install planning-like-coding
```

Restart Claude Code or run `/skills`.

## Trigger

Say "check X across the vault," "audit Y," "scan all Z and compare," "do a health check on...," "gather data from multiple places" — or invoke explicitly: `/planning-like-coding`.

## When to Use

- 3+ independent exploration dimensions
- Scanning/counting across non-overlapping directories or files
- Audits, health checks, cross-cutting comparisons

**Not for**: single-step tasks, tightly-coupled sequential steps, or precise step-by-step instructions.

## Example

[`examples/vault-health-check.md`](examples/vault-health-check.md) — 5 parallel sub-agents checking stale notes, broken links, orphans, tags, and vault size. 5/5 success, 59:1 isolation.

## License

MIT
