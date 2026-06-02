# Vault Health Check — Complete Walkthrough

## Scenario

User request: *"Check my Obsidian vault health — stale notes, broken links, orphan pages, tag consistency, and vault size trends."*

Five independent dimensions. Classic planning-like-coding trigger.

---

## Step 1: Write Pseudocode & Create Trace File

```
main():
    parallel:
        A = check_stale_notes()  → {stale: [{file, days_since_modified}], total_notes: int}
        B = check_broken_links() → {broken: [{source, target, type}], total_links: int}
        C = check_orphans()      → {orphans: [filename], count: int}
        D = check_tags()         → {inconsistent: [{tag, variants}], total_tags: int}
        E = check_vault_size()   → {total_mb: float, top_dirs: [{name, mb}], trend: str}
    return assemble_report(A, B, C, D, E)
```

All five functions are independent → **dispatch in one parallel batch of 5**.

After user confirms, create trace file:

```
F:\Note\Myself\.claude\traces\2026-06-02-143000-vault-health-check.md
```

Initial content:
```markdown
# Trace: Vault health check
**Started**: 2026-06-02 14:30:00
**Status**: running

## Pseudocode
... (full pseudocode, annotated: "All 5 independent → 1 parallel batch")
```

---

## Step 2: User Confirms Pseudocode

Show the pseudocode. User says "go ahead."

---

## Step 3: Dispatch 5 Sub-agents (One Batch, All Parallel)

### Sub-agent A — `check_stale_notes`

```
Goal: Find notes not modified in > 90 days across the vault.

Return format (strict):
{
  "meta": {
    "evidence_count": <int>,
    "source_count": <int>,
    "contradiction_count": <int>
  },
  "data": {
    "stale": [{file: str, days_since_modified: int}, ...],
    "total_notes": int
  }
}

Return only the structured result. No extra explanation.
```

**Returned:**
```json
{
  "meta": {
    "evidence_count": 3,
    "source_count": 342,
    "contradiction_count": 0
  },
  "data": {
    "stale": [
      {"file": "20-Projects/archived-idea.md", "days_since_modified": 213},
      {"file": "30-Knowledge/old-pattern.md", "days_since_modified": 157},
      {"file": "00-Inbox/draft-2025-11.md", "days_since_modified": 104}
    ],
    "total_notes": 342
  }
}
```

**Append to trace file:**
```markdown
### check_stale_notes — completed 14:30:23 (parallel, batch 1)
- **Status**: success
- **Evidence count**: 3 (3 stale notes found)
- **Source count**: 342 (all .md files scanned)
- **Contradiction count**: 0

**Findings**: 3 notes unmodified >90 days out of 342 total:
- `20-Projects/archived-idea.md` — 213 days
- `30-Knowledge/old-pattern.md` — 157 days
- `00-Inbox/draft-2025-11.md` — 104 days
```

---

### Sub-agent B — `check_broken_links`

```
Goal: Find all internal [[wikilinks]] pointing to non-existent files.

Return format (strict):
{
  "meta": {
    "evidence_count": <int>,
    "source_count": <int>,
    "contradiction_count": <int>
  },
  "data": {
    "broken": [{source: str, target: str, type: str}, ...],
    "total_links": int
  }
}

Return only the structured result. No extra explanation.
```

**Returned:**
```json
{
  "meta": {
    "evidence_count": 4,
    "source_count": 342,
    "contradiction_count": 0
  },
  "data": {
    "broken": [
      {"source": "30-Knowledge/routing-index.md", "target": "deleted-page", "type": "wikilink"},
      {"source": "20-Projects/setup-guide.md", "target": "old-config", "type": "wikilink"},
      {"source": "10-Daily/2026-05-28.md", "target": "moved-note", "type": "wikilink"},
      {"source": "50-Person/long-term-memory.md", "target": "missing-ref", "type": "wikilink"}
    ],
    "total_links": 1247
  }
}
```

**Append to trace file:**
```markdown
### check_broken_links — completed 14:30:25 (parallel, batch 1)
- **Status**: success
- **Evidence count**: 4 (broken wikilinks)
- **Source count**: 342 (files scanned for [[ ]])
- **Contradiction count**: 0

**Findings**: 4 broken links out of 1247 total:
- `routing-index.md` → `deleted-page`
- `setup-guide.md` → `old-config`
- `2026-05-28.md` → `moved-note`
- `long-term-memory.md` → `missing-ref`
```

---

### Sub-agent C — `check_orphans`

```
Goal: Find notes with zero incoming links (orphan pages).

Return format (strict):
{
  "meta": {
    "evidence_count": <int>,
    "source_count": <int>,
    "contradiction_count": <int>
  },
  "data": {
    "orphans": [str],
    "count": int
  }
}

Return only the structured result. No extra explanation.
```

**Returned:**
```json
{
  "meta": {
    "evidence_count": 6,
    "source_count": 342,
    "contradiction_count": 0
  },
  "data": {
    "orphans": [
      "00-Inbox/random-thought.md",
      "30-Knowledge/one-off-note.md",
      "30-Knowledge/til/2025-06-15.md",
      "20-Projects/draft-spec.md",
      "00-Inbox/meeting-notes-raw.md",
      "30-Knowledge/forgotten-pattern.md"
    ],
    "count": 6
  }
}
```

**Append to trace file:**
```markdown
### check_orphans — completed 14:30:27 (parallel, batch 1)
- **Status**: success
- **Evidence count**: 6 (orphan pages)
- **Source count**: 342 (notes checked for incoming links)
- **Contradiction count**: 0

**Findings**: 6 orphan pages (zero incoming links):
- `00-Inbox/random-thought.md`
- `30-Knowledge/one-off-note.md`
- `30-Knowledge/til/2025-06-15.md`
- `20-Projects/draft-spec.md`
- `00-Inbox/meeting-notes-raw.md`
- `30-Knowledge/forgotten-pattern.md`
```

---

### Sub-agent D — `check_tags`

```
Goal: Scan all frontmatter tags. Find inconsistent casing or near-duplicate variants (e.g. #React vs #react).

Return format (strict):
{
  "meta": {
    "evidence_count": <int>,
    "source_count": <int>,
    "contradiction_count": <int>
  },
  "data": {
    "inconsistent": [{tag: str, variants: [str]}, ...],
    "total_tags": int
  }
}

Return only the structured result. No extra explanation.
```

**Returned:**
```json
{
  "meta": {
    "evidence_count": 3,
    "source_count": 89,
    "contradiction_count": 0
  },
  "data": {
    "inconsistent": [
      {"tag": "cli", "variants": ["#CLI", "#cli", "#Cli"]},
      {"tag": "typescript", "variants": ["#TypeScript", "#typescript"]},
      {"tag": "obsidian", "variants": ["#Obsidian", "#obsidian"]}
    ],
    "total_tags": 89
  }
}
```

**Append to trace file:**
```markdown
### check_tags — completed 14:30:28 (parallel, batch 1)
- **Status**: success
- **Evidence count**: 3 (inconsistent tag groups)
- **Source count**: 89 (unique tags scanned)
- **Contradiction count**: 0

**Findings**: 3 tag groups with inconsistent casing out of 89 unique tags:
- `cli`: #CLI / #cli / #Cli
- `typescript`: #TypeScript / #typescript
- `obsidian`: #Obsidian / #obsidian
```

---

### Sub-agent E — `check_vault_size`

```
Goal: Measure vault disk usage: total size, top-5 directories by size, month-over-month trend from git history or file timestamps.

Return format (strict):
{
  "meta": {
    "evidence_count": <int>,
    "source_count": <int>,
    "contradiction_count": <int>
  },
  "data": {
    "total_mb": float,
    "top_dirs": [{name: str, mb: float}, ...],
    "trend": str
  }
}

Return only the structured result. No extra explanation.
```

**Returned:**
```json
{
  "meta": {
    "evidence_count": 5,
    "source_count": 5,
    "contradiction_count": 0
  },
  "data": {
    "total_mb": 184.3,
    "top_dirs": [
      {"name": "30-Knowledge", "mb": 72.1},
      {"name": "20-Projects", "mb": 48.5},
      {"name": "10-Daily", "mb": 31.2},
      {"name": "50-Person", "mb": 18.9},
      {"name": "00-Inbox", "mb": 13.6}
    ],
    "trend": "+8.2 MB/month over last 3 months"
  }
}
```

**Append to trace file:**
```markdown
### check_vault_size — completed 14:30:30 (parallel, batch 1)
- **Status**: success
- **Evidence count**: 5 (top-level directories measured)
- **Source count**: 5 (directories analyzed)
- **Contradiction count**: 0

**Findings**: 184.3 MB total, +8.2 MB/month trend.
Top directories: 30-Knowledge (72.1 MB), 20-Projects (48.5 MB), 10-Daily (31.2 MB), 50-Person (18.9 MB), 00-Inbox (13.6 MB)
```

---

## Step 4: Assemble Results

Main agent merges the 5 JSON payloads into a single report:

```
## Vault Health Report — 2026-06-02

### Overview
- 342 notes, 1247 links, 184.3 MB
- Growth trend: +8.2 MB/month

### Stale Notes (3)
- 20-Projects/archived-idea.md — 213 days
- 30-Knowledge/old-pattern.md — 157 days
- 00-Inbox/draft-2025-11.md — 104 days

### Broken Links (4)
- routing-index.md → deleted-page
- setup-guide.md → old-config
- 2026-05-28.md → moved-note
- long-term-memory.md → missing-ref

### Orphan Pages (6)
- 00-Inbox/random-thought.md
- 30-Knowledge/one-off-note.md
- 30-Knowledge/til/2025-06-15.md
- 20-Projects/draft-spec.md
- 00-Inbox/meeting-notes-raw.md
- 30-Knowledge/forgotten-pattern.md

### Tag Inconsistencies (3)
- cli: #CLI / #cli / #Cli
- typescript: #TypeScript / #typescript
- obsidian: #Obsidian / #obsidian

### Actions
1. Review 3 stale notes — archive or refresh
2. Fix 4 broken links — update or remove targets
3. Decide fate of 6 orphans — link or delete
4. Normalize 3 tag groups to single casing
```

**Final trace file append:**

```markdown
## Assembled Result
(complete report above)

## Execution Summary

| Metric | Value |
|--------|-------|
| Total functions | 5 |
| Parallel batches | 1 (all 5 ran concurrently) |
| Serial steps | 0 |
| Succeeded / failed | 5 / 0 |
| Cross-function contradictions | 0 |
| Trace file | `.claude/traces/2026-06-02-143000-vault-health-check.md` |
```

---

## Stats

| Metric | Value |
|--------|-------|
| Total sub-agents | 5 |
| First-try success | 5/5 |
| Context isolation ratio | 59:1 |
| Main agent tokens | ~1,200 |
| Sub-agent tokens (total, invisible to main) | ~71,000 |
| Time saved vs. sequential | ~4x |
| Trace file written to | `.claude/traces/2026-06-02-143000-vault-health-check.md` |

**59:1** means for every token the main agent saw, 59 tokens of search traces, file reads, and intermediate reasoning stayed inside sub-agent contexts — completely invisible to the main agent. The main agent only received the 5 JSON payloads (~200 tokens each).

The trace file gives the user a live, human-readable record of every function call — what ran, in what order (parallel vs serial), how much evidence backs each result, and where contradictions were found. The model never reads this file; it's purely for the user to inspect progress and audit results.
