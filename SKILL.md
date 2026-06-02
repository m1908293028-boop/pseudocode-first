---
name: planning-like-coding
description: >-
  Multi-dimensional analysis via isolated sub-agents: write pseudocode, dispatch
  functions as sub-agents, assemble structured returns. Use when the task needs
  3+ independent searches/scans/audits across different directories, keywords,
  or dimensions. Not for single-step tasks or tightly-coupled sequential steps.
user-invocable: true
disable-model-invocation: false
when_to_use: >-
  Triggered by: "check X across the vault", "audit Y", "scan all Z and compare",
  "find everything about A, B, and C", "do a health check on...", "gather data
  from multiple places", or any request implying breadth (multiple directories /
  keywords / dimensions). Prefer this over direct Grep/Bash chains when there are
  3+ independent search targets or structured assembly is required.
allowed-tools: Agent, Read, Write, Bash, Grep, Glob
---

# Planning Like Coding

Treat agent orchestration like writing a program. The main agent is `main()`. Sub-agents are function calls. Context isolation = scope.

## Trace File

Every invocation writes a **trace file** — a human-readable log of what ran, in what order, and what each function found. The model **never reads this file**. It exists only for the user to inspect progress and audit results.

**Location**: `<project-root>/.claude/traces/YYYY-MM-DD-HHmmss-<task-slug>.md`

If `.claude/traces/` doesn't exist, create it (on Unix: `mkdir -p`; on Windows: `mkdir` handles intermediate directories natively). If there is no clear project root (e.g. working from `~`), fall back to `./.claude/traces/...` relative to the current working directory.

**Full structure** (written incrementally across the workflow):

```
# Trace: <one-line task summary>
**Started**: YYYY-MM-DD HH:MM:SS
**Status**: running

## Pseudocode
(full pseudocode block with parallel/serial annotations)

## Execution Log

### func_X — completed HH:MM:SS (parallel|serial, batch N)
- **Status**: success | failed | partial
- **Evidence**: N data points found
- **Sources**: N files/URLs consulted
- **Contradictions**: N (within this function's own data)

**Findings**:
(human-readable summary — bullet lists for file paths, inline text for short strings)

## Assembled Result
(the final output shown to the user)

## Execution Summary
(table of aggregate stats — see Step 4)
```

**When to write each section:**

| Event | Action |
|-------|--------|
| Pseudocode confirmed | Create file; write `# Trace` header, status, `## Pseudocode`, `## Execution Log` header |
| Sub-agent returns | Append `### func_X` entry under Execution Log |
| Assembly done | Append `## Assembled Result` + `## Execution Summary`; edit top to set status `complete` |
| Aborted mid-execution | Edit top to set status `aborted`; append a note under Execution Log listing what failed |

## Workflow

### 1. Write pseudocode

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

func_C(dependency):
    Goal: one sentence; depends on result_A.field1
    Returns: {field5: type}
```

Execution strategy:
- **No data dependency** → same parallel batch
- **Depends on one prior result** → serial, after that function completes
- **Depends on multiple prior results** → serial, after the LAST dependency completes (note in pseudocode: `// blocks on: A, B`)
- **Max 5 per parallel batch** → split into successive batches in definition order

### 2. User confirms pseudocode

Show the pseudocode. Wait for explicit approval before dispatching.

If the user has said "just go ahead" or "don't ask for confirmation", skip confirmation for this session but still write pseudocode internally — it forces decomposition before action.

If the user modifies the pseudocode, only adjust the flagged functions. Leave unrelated ones untouched.

**After confirmation**, create the trace file with the header, status `running`, and `## Pseudocode` section.

### 3. Dispatch sub-agents

Each function = one sub-agent (depth = 1). Prompt must include:
- The goal (one clear sentence)
- Input values from prior serial steps, if any
- The expected return format: `{ "meta": {...}, "data": {...} }` JSON

**Return format** (every sub-agent must follow this):

```json
{
  "meta": {
    "evidence_count": <int>,
    "source_count": <int>,
    "contradiction_count": <int>
  },
  "data": { <function-specific fields> }
}
```

| Field | Meaning |
|-------|---------|
| `evidence_count` | How many concrete findings / data points / anomalies discovered |
| `source_count` | How many distinct files, URLs, or data sources were consulted |
| `contradiction_count` | Internal inconsistencies found within this function's own data (0 if none) |

The `data` field holds the function-specific payload. No Markdown, no prose — JSON only, so the main agent merges results without parsing natural language.

**After each sub-agent returns**, append to the trace file under `## Execution Log`:

- Status line (success / failed / partial)
- The three meta numbers with brief parenthetical descriptions
- A **human-readable "Findings" summary** — convert paths to backticked relative links, convert short strings to inline text. Do NOT dump raw JSON here. The user reads this, not a parser.

Example trace entry:
```markdown
### check_stale_notes — completed 14:30:23 (parallel, batch 1)
- **Status**: success
- **Evidence**: 3 (stale notes found)
- **Sources**: 342 (.md files scanned)
- **Contradictions**: 0

**Findings**: 3 notes unmodified >90 days out of 342 total:
- `20-Projects/archived-idea.md` — 213 days
- `30-Knowledge/old-pattern.md` — 157 days
- `00-Inbox/draft-2025-11.md` — 104 days
```

**Sub-agent tools**: Explore agents get only read/search tools (Read, Grep, Glob). General-purpose agents may also use Write, Edit, Bash. Never give a sub-agent tools it doesn't need.

**Parallel limit**: max 5 per batch. If more than 5 independent functions, split into batches of 5 in definition order.

**Error handling**:

| Failure | Action |
|---------|--------|
| Sub-agent times out or crashes | Retry once. If it fails again, reduce scope or fall back to direct tools. In trace: `**Status**: failed`, note the error. |
| Sub-agent returns prose instead of JSON | Re-prompt once with format emphasized. If still unparseable, extract what you can, note the gap, proceed. In trace: `**Status**: partial`. |
| Sub-agent returns malformed but mostly valid JSON | Extract usable fields manually; annotate recovered vs. missing in trace. |
| Sub-agent returns empty / "not found" | Spot-check with a direct Grep. If confirmed empty, trust it. In trace: `Evidence: 0`, note "confirmed empty via spot-check". |
| Two sub-agents contradict each other | Do NOT re-run either. Resolve with one direct check (Grep/Bash). Append a `### resolution` note under Execution Log describing the conflict and resolution. |
| All sub-agents fail | Abort. Report to user, suggest narrowing scope. Set trace status to `aborted`. |

### 4. Assemble results

Combine return values per the pseudocode's `assemble()` logic. Never inspect sub-agent internal context.

After assembly, finalize the trace file:
1. Append `## Assembled Result` with the final output
2. Append `## Execution Summary`:

```markdown
| Metric | Value |
|--------|-------|
| Total functions | N |
| Parallel batches | N |
| Serial steps | N |
| Succeeded / failed | N / N |
| Contradictions found (cross-function) | N |
| Trace file | (relative path) |
```

Do NOT sum `evidence_count` or `source_count` across functions — evidence from "3 broken links" and "184.3 MB disk usage" are different dimensions and a total is meaningless. Report per-function in the Assembled Result instead.
3. Update the top-of-file status line from `running` to `complete`.

## Sub-agent Prompt Template

```
Goal: {func_X's goal}

Input:
{concrete parameters from main agent, if any}

Return format (strict):
{
  "meta": {
    "evidence_count": <int>,
    "source_count": <int>,
    "contradiction_count": <int>
  },
  "data": { <fields> }
}

Return only the structured JSON. No extra explanation.
```

## Context Passing

Main agent and sub-agent contexts are physically isolated. **Never read a file ≥2KB just to pass it to a sub-agent** — that shifts cost without saving tokens.

| Situation | Action |
|-----------|--------|
| Main agent already read the file (sunk cost) | Pass content — saves re-reading |
| Main agent hasn't read it | Don't fetch it. Let sub-agent search independently |
| Main agent can describe from memory | Pass brief description for targeted search |

Exception: if the file is small (<2KB) and the sub-agent would need an expensive search to locate it, reading to pass can be a net win.

## When to Use

- 3+ independent exploration dimensions
- Scanning/counting across non-overlapping directories or files
- Collecting data from multiple sources, single assembly step
- Requests implying "find everything about X, Y, and Z"

## When NOT to Use

- Single-step tasks or tightly-coupled sequential steps (each doable with one Bash/Grep)
- User gave precise step-by-step instructions — follow them directly
- The only reason to read a file is to pass it to a sub-agent (anti-pattern: cost shift, not savings)

## Example

See `examples/vault-health-check.md` — 5 parallel sub-agents, 59:1 context isolation, full trace file walkthrough.
