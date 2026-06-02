---
name: planning-like-coding
description: >-
  Use whenever the user asks to analyze, audit, research, investigate, scan,
  compare, or check multiple things. Triggers on tasks that involve collecting
  data from 3+ independent sources and synthesizing a summary. Write natural-language
  pseudocode first to decompose into functions, then execute each function as an
  isolated sub-agent. Sub-agents return only structured results — their internal
  context (search traces, intermediate reads, reasoning) never enters the main
  agent context. This is the default approach for any multi-dimensional analysis
  task unless the user specifies otherwise.
user-invocable: true
disable-model-invocation: false
when_to_use: >-
  Also trigger when the user says "check X across the vault", "audit Y",
  "scan all Z and compare", "find everything about A, B, and C", "do a
  health check on...", "gather data from multiple places", or any request
  that implies breadth (multiple directories / keywords / dimensions).
  Prefer this skill over direct Grep/Bash chains when there are 3+ independent
  search targets or when structured assembly is required.
allowed-tools: Agent, Read, Write, Edit, Bash, Grep, Glob
---

# Planning Like Coding

Treat agent orchestration like writing a program. The main agent is `main()`. Sub-agents are function calls. Context isolation = scope.

## Core Analogy

| Code | Agent |
|------|-------|
| Function signature | Pseudocode: goal + input + return type |
| Arguments | Main agent passes task description + expected output format to sub-agent |
| Function body | Sub-agent's internal search / read / reasoning |
| Local variables | Sub-agent's context window (invisible to parent) |
| Return value | Structured result matching the format main agent defined |
| `main()` assembly | Main agent combines return values into final output |

## Workflow

### 1. Write pseudocode first

Do NOT start executing. Output the plan:

```
main():
    parallel:
        result_A = func_A(input) → {field1: [items], field2: int}
        result_B = func_B(input) → [{field3: type, field4: type}]
    serial:
        result_C = func_C(result_A.field1) → {field5: type}
    return assemble(result_A, result_B, result_C)

func_A(input):
    Goal: one sentence describing what this function does
    Returns: {field1: [items], field2: int}

func_B(input):
    Goal: one sentence describing what this function does
    Returns: [{field3: type, field4: type}, ...]
```

- No data dependency → parallel
- Data dependency → serial

### 2. User confirms pseudocode

Show the pseudocode to the user. Wait for explicit approval before dispatching sub-agents.

If the user says "just go ahead" or "don't ask for confirmation", skip confirmation for this session but still write pseudocode internally — it forces you to think through decomposition before acting.

If the user rejects or modifies the pseudocode, only adjust the functions that were flagged. Leave unrelated functions untouched.

### 3. Dispatch sub-agents

Each function = one sub-agent. Sub-agents never spawn other sub-agents (depth = 1). Sub-agent prompt must include:

- The goal (one clear sentence)
- The expected return format as JSON (fields, types, example structure)
- Any input values from prior serial steps

All sub-agent returns must be JSON. No Markdown tables, no prose summaries — JSON only, so the main agent can merge results without parsing natural language.

**Parallel limit**: max 5 sub-agents per batch. If more than 5 functions are independent, batch them in definition order (first 5, then next 5, etc.).

**Sub-agent tools**: Explore agents get only read/search tools (Read, Grep, Glob). General-purpose agents may also use Write, Edit, Bash. Never give a sub-agent tools it doesn't need for its specific function.

**Error handling**:

| Failure | Action |
|---------|--------|
| Sub-agent times out or crashes | Retry once with the same prompt. If it fails again, reduce scope or fall back to direct tools |
| Sub-agent returns unparseable format (e.g. prose instead of JSON) | Re-prompt once with format emphasized. If still unparseable, note the gap and proceed with other results |
| Sub-agent returns mostly valid but slightly malformed JSON | Extract usable fields manually, annotate which fields were recovered vs. missing |
| Sub-agent returns empty / "not found" | Spot-check with a quick direct Grep to rule out wrong-directory errors. If confirmed empty, trust it and note in assembly |
| Results from two sub-agents contradict each other | Main agent resolves with a single direct check (Grep/Bash). Do not re-run either sub-agent |
| All sub-agents fail | Abort the batch. Report to user, suggest narrowing scope |

### 4. Assemble results

Main agent combines return values per the pseudocode's `assemble()` logic. Never inspect sub-agent internal context.

## Context Passing Strategy

Main agent and sub-agent contexts are physically isolated. Files the main agent read do NOT automatically appear in sub-agent context.

**Rule: Never read a large file just to pass it to a sub-agent.** That shifts cost without saving tokens. Exception: if the file is small (<2KB, ~500 tokens) and the sub-agent would need an expensive search to locate it, reading to pass can be a net win.

| Situation | Action |
|-----------|--------|
| Main agent already read the file (sunk cost for another reason) | Pass content as sub-agent input — saves re-reading |
| Main agent hasn't read it, would need to fetch it | Don't pass. Let sub-agent search independently |
| Main agent can describe from memory | Pass brief description for targeted search |

When `inherit_context` becomes available in Claude Code, main agent can zero-cost share already-loaded context with sub-agents. Until then, follow the table above.

## Sub-agent Prompt Template

```
Goal: {func_X's goal}

Input:
{concrete parameters / context from main agent, if any}

Return format (strict):
{field: type // description}

Return only the structured result. No extra explanation.
```

## When to Use

- Task has 3+ independent exploration dimensions
- Scanning / counting across non-overlapping directories or files
- Collecting data from multiple sources, single assembly step
- User asks to audit, health-check, investigate, or compare across breadth
- Any request that implies "find everything about X, Y, and Z"

## Example

See `examples/vault-health-check.md` for a complete walkthrough: pseudocode → 5 parallel sub-agents → assembly. 59:1 context isolation ratio, 5/5 sub-agents first-try success.

## When NOT to Use

- Single-step tasks
- Steps are strongly dependent AND each step can be done with a single Bash/Grep command
- User gave precise step-by-step instructions
- Main agent would need to fetch data solely to pass to sub-agents (cost shift, not savings)

### Anti-pattern

```
// WRONG: main agent reads a 50KB file just to pass it to a sub-agent
//        → 25000 tokens wasted in main context for zero net savings
main():
    big_file = read("huge-report.md")    // DON'T
    result = func_analyze(big_file)      // Cost shifted, not saved
```
The correct approach: let the sub-agent read `huge-report.md` inside its own isolated context. Main agent never sees those 25000 tokens.
