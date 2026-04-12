---
name: index-knowledge
description: Generate a hierarchical AGENTS.md knowledge base for any codebase. Analyzes project structure, scoring directories by complexity and domain distinctness, then produces root and subdirectory AGENTS.md files concise enough for LLM context windows. Triggers on requests to "index", "document", "map", or "generate knowledge" for a codebase, or when onboarding onto a new project.
license: MIT
metadata:
  author: mynameistito, dmmulroy
  version: "1.1.0"
---

# index-knowledge

Generate hierarchical AGENTS.md files — a single root overview plus targeted subdirectory docs scored by complexity and domain distinctness. Every file stays under 150 lines so it fits in an LLM context window without drowning the model in boilerplate.

**This skill is OS-agnostic.** All structural analysis uses agent-native tools (glob, grep, read) that work on macOS, Linux, WSL, Windows PowerShell, and Windows CMD. No platform-specific shell commands required. If `rg` (ripgrep) is available, it can be used for faster content search by invoking it through the runtime's process/spawn facilities (e.g., Node.js `child_process.spawnSync`, Python `subprocess.run`, Go `exec.Command`) — never hardcode a shell like bash or PowerShell.

## Abstract

Most codebases lack machine-readable orientation: no quick map of where things live, what conventions are non-obvious, or which directories are complexity hotspots. `index-knowledge` fixes that by producing a hierarchy of concise AGENTS.md files. It discovery-scans the tree in parallel (agent-native structural analysis + explore agents + LSP symbols where available), scores each directory on file count, symbol density, module boundaries, and export centrality, then decides which locations warrant their own doc. Root gets the full treatment (`OVERVIEW`, `STRUCTURE`, `WHERE TO LOOK`, `CODE MAP`, `CONVENTIONS`, `ANTI-PATTERNS`, `COMMANDS`, `NOTES`); subdirectories get a leaner version that never repeats the parent. The result is a map an agent can follow in seconds instead of minutes.

<rg-preference>
**Check for `rg` (ripgrep) availability early — use it if present, otherwise default to `grep`.** At the start of Phase 1, attempt to run `rg --version` using the runtime's non-shell process/spawn facilities (e.g., Node.js `child_process.spawnSync`, Python `subprocess.run` or `shutil.which`, Go `exec.LookPath` / `exec.Command`) without invoking a shell. If that subprocess invocation returns successfully, prefer `rg` for:
- File listing: `rg --files -g 'glob' path` (respects .gitignore and rg ignore rules)
- Content search: `rg pattern path -g 'glob'`
- File listing by content: `rg -l pattern path`

For **total line counts**, do not use `rg --count` or `rg -c` (they count only matching/non-empty lines, skipping blanks). Instead: use `rg --files` to produce the file list (preserving ignore rules), then count physical lines per file with a line-counting utility and sum the totals.

If `rg` is not installed, use the agent-native `grep` tool as the default. Always have a `grep` fallback — never require `rg`.

`rg` is cross-platform (Windows/macOS/Linux) and significantly faster than grep on large codebases.
</rg-preference>

## Usage

```text
--create-new   # Read existing → remove all → regenerate from scratch
--max-depth=2  # Limit directory depth (default: 5)
```

Default: Update mode (modify existing + create new where warranted)

---

## Workflow (High-Level)

1. **Discovery + Analysis** (concurrent)
   - Launch parallel explore agents (multiple Task calls in one message)
   - Main session: agent-native structure analysis + LSP codemap + read existing AGENTS.md
2. **Score & Decide** - Determine AGENTS.md locations from merged findings
3. **Generate** - Root first, then subdirs in parallel
4. **Review** - Deduplicate, trim, validate
5. **CLAUDE.md Bridge** - Create CLAUDE.md alongside every AGENTS.md location

<critical>
**TodoWrite ALL phases. Mark in_progress → completed in real-time.**

```text
TodoWrite([
  { id: "discovery", content: "Fire explore agents + LSP codemap + read existing", status: "pending", priority: "high" },
  { id: "scoring", content: "Score directories, determine locations", status: "pending", priority: "high" },
  { id: "generate", content: "Generate AGENTS.md files (root + subdirs)", status: "pending", priority: "high" },
  { id: "review", content: "Deduplicate, validate, trim", status: "pending", priority: "medium" },
  { id: "claude-md", content: "Create CLAUDE.md bridge files alongside AGENTS.md locations", status: "pending", priority: "medium" }
])
```
</critical>

---

## Phase 1: Discovery + Analysis (Concurrent)

**Mark "discovery" as in_progress.**

### Launch Parallel Explore Agents

Multiple Task calls in a single message execute in parallel. Results return directly.

```text
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
**DYNAMIC AGENT SPAWNING**: After structural analysis, spawn ADDITIONAL explore agents based on project scale:

| Factor | Threshold | Additional Agents |
|--------|-----------|-------------------|
| **Total files** | >100 | +1 per 100 files |
| **Total lines** | >10k | +1 per 10k lines |
| **Directory depth** | ≥4 | +2 for deep exploration |
| **Large files (>500 lines)** | >10 files | +1 for complexity hotspots |
| **Monorepo** | detected | +1 per package/workspace |
| **Multiple languages** | >1 | +1 per language |

Use agent-native tools to measure project scale. Read the project root directory, then use glob to count files by pattern. Replace `{{PROJECT_ROOT}}` with the actual absolute path to the project:

```text
// Count source files — adapt patterns to detected languages
glob(pattern="**/*.{ts,tsx,js,py,go,rs}", path="{{PROJECT_ROOT}}")

// Discover project directories (read root listing to find src, lib, app, cmd, pkg, etc.)
read(filePath="{{PROJECT_ROOT}}")

// Then glob each discovered source directory for file counts
glob(pattern="src/**", path="{{PROJECT_ROOT}}")
glob(pattern="lib/**", path="{{PROJECT_ROOT}}")
// ... adapt to the actual directories found above

// Check directory depth by reading subdirectories of EACH discovered source directory
// Do NOT hardcode "src" — use the directories found above
// e.g. if root listing reveals src/, lib/, app/ → read each one:
read(filePath="{{PROJECT_ROOT}}/{discovered_dir}")  // lists entries
```

From glob results, derive:
- **total_files**: count from the language-extension glob (covers all source dirs regardless of layout)
- **total_lines**: use `grep` with line counting or `read` files and count; if `rg` is available, use `rg --files` to list source files (respecting ignore rules), then count physical lines per file and sum (do not use `rg --count`, which skips blank lines)
- **large_files**: files where read reveals >500 lines
- **max_depth**: deepest directory nesting level

Example spawning (all in ONE message for parallel execution):

```text
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

#### 1. Structural Analysis (OS-Agnostic)

Use agent-native tools for all structural discovery. These work identically on macOS, Linux, WSL, Windows PowerShell, and Windows CMD.

**Directory depth + file counts:**

```text
// List directory tree from root
read(filePath="{{PROJECT_ROOT}}")

// Then recursively read subdirectories to map depth
// Use directories discovered from root listing — never assume "src" exists
read(filePath="{{PROJECT_ROOT}}/{discovered_dir_1}")
read(filePath="{{PROJECT_ROOT}}/{discovered_dir_2}")
// ... etc for each major directory found above
```

**Files per directory (top 30):**

```text
// Use glob to count files per directory
// Use glob on EACH discovered source directory (never hardcode "src")
glob(pattern="{discovered_dir}/**/*", path="{{PROJECT_ROOT}}")
glob(pattern="{discovered_dir}/**/*", path="{{PROJECT_ROOT}}")

// For aggregate counts, use grep to find all files matching extensions
// Search the project root recursively instead of a single "src" dir
// If rg was detected, prefer it via the cross-platform process invocation
// (see <rg-preference> above) — e.g. spawn("rg", ["--files", "-g", "*.{ts,tsx,js,py,go,rs}", "{{PROJECT_ROOT}}"])
grep(pattern=".", glob="*.{ts,tsx,js,py,go,rs}", path="{{PROJECT_ROOT}}")
```

**Code concentration by extension:**

```text
// Find all source files by extension
glob(pattern="**/*.py", path="{{PROJECT_ROOT}}")
glob(pattern="**/*.ts", path="{{PROJECT_ROOT}}")
glob(pattern="**/*.tsx", path="{{PROJECT_ROOT}}")
glob(pattern="**/*.js", path="{{PROJECT_ROOT}}")
glob(pattern="**/*.go", path="{{PROJECT_ROOT}}")
glob(pattern="**/*.rs", path="{{PROJECT_ROOT}}")
```

**Existing AGENTS.md / CLAUDE.md:**

```text
glob(pattern="**/AGENTS.md", path="{{PROJECT_ROOT}}")
glob(pattern="**/CLAUDE.md", path="{{PROJECT_ROOT}}")
```

<critical>
**Use absolute paths** — all glob, grep, read, and rg (when invoked via subprocess) calls should use absolute file paths. Derive the project root from the working directory context and construct paths from there.
</critical>

#### 2. Read Existing AGENTS.md

```text
For each existing file found:
  Read(filePath=file)
  Extract: key insights, conventions, anti-patterns
  Store in EXISTING_AGENTS map
```

If `--create-new`: Read all existing first (preserve context) → then delete all → regenerate.

#### 3. LSP Codemap (if available)

```text
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

**LSP Fallback**: If unavailable, rely on explore agents + ast-grep. If `rg` is available, prefer it over grep for content search.

**Merge: structural analysis + LSP + existing + Task agent results. Mark "discovery" as completed.**

---

## Phase 2: Scoring & Location Decision

**Mark "scoring" as in_progress.**

### Scoring Matrix

| Factor | Weight | High Threshold | Source |
|--------|--------|----------------|--------|
| File count | 3x | >20 | glob |
| Subdir count | 2x | >5 | read |
| Code ratio | 2x | >70% | glob |
| Unique patterns | 1x | Has own config | explore |
| Module boundary | 2x | Has index.ts/__init__.py | glob |
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

```text
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
\`\`\`
{dev/test/build — use platform-appropriate syntax}
\`\`\`

## NOTES
{Gotchas}
```

**Quality gates**: 50-150 lines, no generic advice, no obvious info.

### Subdirectory AGENTS.md (Parallel)

Launch general agents for each location in ONE message (parallel execution):

```text
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

## Phase 5: Create CLAUDE.md Bridge Files

**Mark "claude-md" as in_progress.**

For every directory that received an AGENTS.md, create a companion CLAUDE.md that points to it. This ensures Claude Code and other tools that look for CLAUDE.md will discover the AGENTS.md context.

### Content (identical for all locations)

```markdown
This project uses AGENTS.md files for AI context. Read AGENTS.md in this directory for relevant instructions and knowledge.
```

### Implementation

For each entry in `AGENTS_LOCATIONS`, write a CLAUDE.md at the same path. Use `Write` tool for each — all writes can happen in parallel (same message).

### Cleanup orphaned bridge files

After creating CLAUDE.md files, scan for any existing CLAUDE.md files whose content matches the bridge template verbatim but whose directory is NOT in `AGENTS_LOCATIONS`. These are stale bridge files left from a prior run (e.g. after `--create-new` removed an AGENTS.md that no longer clears the scoring threshold). Remove them so tools don't point to an AGENTS.md that no longer exists.

```text
// For each CLAUDE.md found via glob, read it.
// If its content matches the bridge template AND its parent dir is NOT in AGENTS_LOCATIONS:
//   Delete it and log "Removed orphaned CLAUDE.md at {path}"
// If its content does NOT match the bridge template:
//   Leave it alone (user-authored, not a bridge file)
```

**Overwrite guard (deterministic, not judgment-based):** Read existing CLAUDE.md first. Only overwrite if one of the following is true:
- The file is empty.
- The file's full content matches the bridge template string verbatim:
  `This project uses AGENTS.md files for AI context. Read AGENTS.md in this directory for relevant instructions and knowledge.`

If neither condition is met, leave the file untouched and log `"CLAUDE.md already exists — skipped."` This prevents destroying user-authored content.

```text
// Example: if AGENTS_LOCATIONS = [{ path: "." }, { path: "src/hooks" }, { path: "src/api" }]
// Then create:
//   ./CLAUDE.md
//   ./src/hooks/CLAUDE.md
//   ./src/api/CLAUDE.md
```

**Mark "claude-md" as completed.**

---

## Final Report

```text
=== index-knowledge Complete ===

Mode: {update | create-new}

Files:
  ✓ ./AGENTS.md (root, {N} lines)
  ✓ ./CLAUDE.md (bridge)
  ✓ ./src/hooks/AGENTS.md ({N} lines)
  ✓ ./src/hooks/CLAUDE.md (bridge)

Dirs Analyzed: {N}
AGENTS.md Created: {N}
AGENTS.md Updated: {N}
CLAUDE.md Created: {N}

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
- **Platform-specific commands**: MUST use agent-native tools (glob, grep, read) — never assume bash or PowerShell availability. If `rg` is detected as available, prefer it for speed but always have a grep fallback.
