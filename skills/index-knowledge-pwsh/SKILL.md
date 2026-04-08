---
name: index-knowledge-pwsh
description: PowerShell - Generate a hierarchical AGENTS.md knowledge base for any codebase. Analyzes project structure, scoring directories by complexity and domain distinctness, then produces root and subdirectory AGENTS.md files concise enough for LLM context windows. Triggers on requests to "index", "document", "map", or "generate knowledge" for a codebase, or when onboarding onto a new project.
license: MIT
metadata:
  author: mynameistito, dmmulroy
  version: "1.0.0"
---

# index-knowledge

Generate hierarchical AGENTS.md files — a single root overview plus targeted subdirectory docs scored by complexity and domain distinctness. Every file stays under 150 lines so it fits in an LLM context window without drowning the model in boilerplate.

## Abstract

Most codebases lack machine-readable orientation: no quick map of where things live, what conventions are non-obvious, or which directories are complexity hotspots. `index-knowledge` fixes that by producing a hierarchy of concise AGENTS.md files. It discovery-scans the tree in parallel (pwsh structural analysis + explore agents + LSP symbols where available), scores each directory on file count, symbol density, module boundaries, and export centrality, then decides which locations warrant their own doc. Root gets the full treatment (`OVERVIEW`, `STRUCTURE`, `WHERE TO LOOK`, `CODE MAP`, `CONVENTIONS`, `ANTI-PATTERNS`, `COMMANDS`, `NOTES`); subdirectories get a leaner version that never repeats the parent. The result is a map an agent can follow in seconds instead of minutes.

## Usage

```
--create-new   # Read existing → remove all → regenerate from scratch
--max-depth=2  # Limit directory depth (default: 5)
```

Default: Update mode (modify existing + create new where warranted)

---

## Workflow (High-Level)

1. **Discovery + Analysis** (concurrent)
   - Launch parallel explore agents (multiple Task calls in one message)
   - Main session: pwsh structure + LSP codemap + read existing AGENTS.md
2. **Score & Decide** - Determine AGENTS.md locations from merged findings
3. **Generate** - Root first, then subdirs in parallel
4. **Review** - Deduplicate, trim, validate

<critical>
**TodoWrite ALL phases. Mark in_progress → completed in real-time.**
  
```
TodoWrite([
  { id: "discovery", content: "Fire explore agents + LSP codemap + read existing", status: "pending", priority: "high" },
  { id: "scoring", content: "Score directories, determine locations", status: "pending", priority: "high" },
  { id: "generate", content: "Generate AGENTS.md files (root + subdirs)", status: "pending", priority: "high" },
  { id: "review", content: "Deduplicate, validate, trim", status: "pending", priority: "medium" }
])
```
</critical>

---

## Phase 1: Discovery + Analysis (Concurrent)

**Mark "discovery" as in_progress.**

### Launch Parallel Explore Agents

Multiple Task calls in a single message execute in parallel. Results return directly.

```
// All Task calls in ONE message = parallel execution

Task(
  description="project structure",
  subagent_type="explore",
  prompt="Project structure: PREDICT standard patterns for detected language → REPORT deviations only"
)

Task(
  description="entry points",
  subagent_type="explore",
  prompt="Entry points: FIND main files → REPORT non-standard organization"
)

Task(
  description="conventions",
  subagent_type="explore",
  prompt="Conventions: FIND config files (.eslintrc, pyproject.toml, .editorconfig) → REPORT project-specific rules"
)

Task(
  description="anti-patterns",
  subagent_type="explore",
  prompt="Anti-patterns: FIND 'DO NOT', 'NEVER', 'ALWAYS', 'DEPRECATED' comments → LIST forbidden patterns"
)

Task(
  description="build/ci",
  subagent_type="explore",
  prompt="Build/CI: FIND .github/workflows, Makefile → REPORT non-standard patterns"
)

Task(
  description="test patterns",
  subagent_type="explore",
  prompt="Test patterns: FIND test configs, test structure → REPORT unique conventions"
)
```

<dynamic-agents>
**DYNAMIC AGENT SPAWNING**: After pwsh analysis, spawn ADDITIONAL explore agents based on project scale:

| Factor | Threshold | Additional Agents |
|--------|-----------|-------------------|
| **Total files** | >100 | +1 per 100 files |
| **Total lines** | >10k | +1 per 10k lines |
| **Directory depth** | ≥4 | +2 for deep exploration |
| **Large files (>500 lines)** | >10 files | +1 for complexity hotspots |
| **Monorepo** | detected | +1 per package/workspace |
| **Multiple languages** | >1 | +1 per language |

```powershell
# Measure project scale first
$skip = '(\\|/)(\.[^\\/]+|node_modules|venv|dist|build)(\\|/)'
$total_files = (Get-ChildItem -Recurse -File | Where-Object { $_.FullName -notmatch $skip }).Count
$src = Get-ChildItem -Recurse -File | Where-Object { $_.Extension -in '.ts','.tsx','.js','.py','.go','.rs' -and $_.FullName -notmatch $skip }
$total_lines = ($src | ForEach-Object { (Get-Content $_.FullName -ErrorAction SilentlyContinue | Measure-Object -Line).Lines } | Measure-Object -Sum).Sum
$large_files = @($src | Where-Object { (Get-Content $_.FullName -ErrorAction SilentlyContinue | Measure-Object -Line).Lines -gt 500 }).Count
$max_depth = (Get-ChildItem -Recurse -Directory | Where-Object { $_.FullName -notmatch $skip } | ForEach-Object { ($_.FullName -split '[\\/]').Count } | Measure-Object -Maximum).Maximum
```

Example spawning (all in ONE message for parallel execution):
```
// 500 files, 50k lines, depth 6, 15 large files → spawn additional agents
Task(
  description="large files",
  subagent_type="explore",
  prompt="Large file analysis: FIND files >500 lines, REPORT complexity hotspots"
)

Task(
  description="deep modules",
  subagent_type="explore",
  prompt="Deep modules at depth 4+: FIND hidden patterns, internal conventions"
)

Task(
  description="cross-cutting",
  subagent_type="explore",
  prompt="Cross-cutting concerns: FIND shared utilities across directories"
)
// ... more based on calculation
```
</dynamic-agents>

### Main Session: Concurrent Analysis

**While Task agents execute**, main session does:

#### 1. Pwsh Structural Analysis
```powershell
# Directory depth + file counts
$skip = '(\\|/)(\.[^\\/]+|node_modules|venv|dist|build)(\\|/)'
Get-ChildItem -Recurse -Directory | Where-Object { $_.FullName -notmatch $skip } | Group-Object { ($_.FullName -split '[\\/]').Count } | Sort-Object Name | Format-Table Count, Name

# Files per directory (top 30)
Get-ChildItem -Recurse -File | Where-Object { $_.FullName -notmatch $skip } | Group-Object { Split-Path $_.FullName -Parent } | Sort-Object Count -Descending | Select-Object -First 30 | Format-Table Count, Name

# Code concentration by extension
Get-ChildItem -Recurse -File | Where-Object { $_.Extension -in '.py','.ts','.tsx','.js','.go','.rs' -and $_.FullName -notmatch $skip } | Group-Object { Split-Path $_.FullName -Parent } | Sort-Object Count -Descending | Select-Object -First 20 | Format-Table Count, Name

# Existing AGENTS.md / CLAUDE.md
Get-ChildItem -Recurse -File -Include 'AGENTS.md','CLAUDE.md' | Where-Object { $_.FullName -notmatch $skip }
```

#### 2. Read Existing AGENTS.md
```
For each existing file found:
  Read(filePath=file)
  Extract: key insights, conventions, anti-patterns
  Store in EXISTING_AGENTS map
```

If `--create-new`: Read all existing first (preserve context) → then delete all → regenerate.

#### 3. LSP Codemap (if available)
```
lsp_servers()  # Check availability

# Entry points — use paths discovered by explore agents (parallel)
lsp_document_symbols(filePath="{discovered_entry_point_1}")
lsp_document_symbols(filePath="{discovered_entry_point_2}")

# Key symbols (parallel)
lsp_workspace_symbols(filePath=".", query="class")
lsp_workspace_symbols(filePath=".", query="interface")
lsp_workspace_symbols(filePath=".", query="function")

# Centrality for top exports
lsp_find_references(filePath="...", line=X, character=Y)
```

**LSP Fallback**: If unavailable, rely on explore agents + AST-grep.

**Merge: pwsh + LSP + existing + Task agent results. Mark "discovery" as completed.**

---

## Phase 2: Scoring & Location Decision

**Mark "scoring" as in_progress.**

### Scoring Matrix

| Factor | Weight | High Threshold | Source |
|--------|--------|----------------|--------|
| File count | 3x | >20 | pwsh |
| Subdir count | 2x | >5 | pwsh |
| Code ratio | 2x | >70% | pwsh |
| Unique patterns | 1x | Has own config | explore |
| Module boundary | 2x | Has index.ts/__init__.py | pwsh |
| Symbol density | 2x | >30 symbols | LSP |
| Export count | 2x | >10 exports | LSP |
| Reference centrality | 3x | >20 refs | LSP |

### Decision Rules

| Score | Action |
|-------|--------|
| **Root (.)** | ALWAYS create |
| **>15** | Create AGENTS.md |
| **8-15** | Create if distinct domain |
| **<8** | Skip (parent covers) |

### Output
```
AGENTS_LOCATIONS = [
  { path: ".", type: "root" },
  { path: "src/hooks", score: 18, reason: "high complexity" },
  { path: "src/api", score: 12, reason: "distinct domain" }
]
```

**Mark "scoring" as completed.**

---

## Phase 3: Generate AGENTS.md

**Mark "generate" as in_progress.**

### Root AGENTS.md (Full Treatment)

```markdown
# PROJECT KNOWLEDGE BASE

**Generated:** {TIMESTAMP}
**Commit:** {SHORT_SHA}
**Branch:** {BRANCH}

## OVERVIEW
{1-2 sentences: what + core stack}

## STRUCTURE
\`\`\`
{root}/
├── {dir}/    # {non-obvious purpose only}
└── {entry}
\`\`\`

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|

## CODE MAP
{From LSP - skip if unavailable or project <10 files}

| Symbol | Type | Location | Refs | Role |

## CONVENTIONS
{ONLY deviations from standard}

## ANTI-PATTERNS (THIS PROJECT)
{Explicitly forbidden here}

## UNIQUE STYLES
{Project-specific}

## COMMANDS
\`\`\`pwsh
{dev/test/build}
\`\`\`

## NOTES
{Gotchas}
```

**Quality gates**: 50-150 lines, no generic advice, no obvious info.

### Subdirectory AGENTS.md (Parallel)

Launch general agents for each location in ONE message (parallel execution):

```
// All in single message = parallel
Task(
  description="AGENTS.md for src/hooks",
  subagent_type="general",
  prompt="Generate AGENTS.md for: src/hooks
    - Reason: high complexity
    - 50-150 lines max
    - NEVER repeat parent content
    - Sections: OVERVIEW (1 line), STRUCTURE (if >5 subdirs), WHERE TO LOOK, CONVENTIONS (if different), ANTI-PATTERNS
    - Write directly to src/hooks/AGENTS.md"
)

Task(
  description="AGENTS.md for src/api",
  subagent_type="general",
  prompt="Generate AGENTS.md for: src/api
    - Reason: distinct domain
    - 50-150 lines max
    - NEVER repeat parent content
    - Sections: OVERVIEW (1 line), STRUCTURE (if >5 subdirs), WHERE TO LOOK, CONVENTIONS (if different), ANTI-PATTERNS
    - Write directly to src/api/AGENTS.md"
)
// ... one Task per AGENTS_LOCATIONS entry
```

**Results return directly. Mark "generate" as completed.**

---

## Phase 4: Review & Deduplicate

**Mark "review" as in_progress.**

For each generated file:
- Remove generic advice
- Remove parent duplicates
- Trim to size limits
- Verify telegraphic style

**Mark "review" as completed.**

---

## Final Report

```
=== index-knowledge Complete ===

Mode: {update | create-new}

Files:
  ✓ ./AGENTS.md (root, {N} lines)
  ✓ ./src/hooks/AGENTS.md ({N} lines)

Dirs Analyzed: {N}
AGENTS.md Created: {N}
AGENTS.md Updated: {N}

Hierarchy:
  ./AGENTS.md
  └── src/hooks/AGENTS.md
```

---

## Anti-Patterns

- **Static agent count**: MUST vary agents based on project size/depth
- **Sequential execution**: MUST parallel (multiple Task calls in one message)
- **Ignoring existing**: ALWAYS read existing first, even with --create-new
- **Over-documenting**: Not every dir needs AGENTS.md
- **Redundancy**: Child never repeats parent
- **Generic content**: Remove anything that applies to ALL projects
- **Verbose style**: Telegraphic or die
