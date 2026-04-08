# index-knowledge

> Generate hierarchical AGENTS.md knowledge bases for any codebase — concise enough for LLM context windows, specific enough to be useful.

Originally created by [@dmmulroy](https://github.com/dmmulroy/) in his [.dotfiles](https://github.com/dmmulroy/.dotfiles) repository. This repo forks and extends the skill with two platform-specific variants.

---

## What It Does

Most codebases lack machine-readable orientation — no quick map of where things live, what conventions matter, or which directories are complexity hotspots. `index-knowledge` fixes that by producing a hierarchy of concise `AGENTS.md` files:

1. **Discovery** — Scans the project tree in parallel using explore agents, shell structural analysis, and LSP symbols
2. **Scoring** — Rates each directory by file count, symbol density, module boundaries, and export centrality
3. **Generation** — Produces a root `AGENTS.md` (full treatment) and targeted subdirectory docs (leaner)
4. **Review** — Deduplicates, trims, and validates output against quality gates

Every generated file stays under 150 lines. No generic advice. No obvious info. Telegraphic style.

### Root AGENTS.md (Full Treatment)

Root files include: `OVERVIEW`, `STRUCTURE`, `WHERE TO LOOK`, `CODE MAP`, `CONVENTIONS`, `ANTI-PATTERNS`, `COMMANDS`, `NOTES`.

### Subdirectory AGENTS.md (Lean)

Subdirectory files include: `OVERVIEW`, `WHERE TO LOOK`, `CONVENTIONS`, `ANTI-PATTERNS` — never repeating parent content.

---

## Variants

This repo provides two platform-specific variants of the same skill. Choose the one that matches your shell environment:

| Variant | Directory | Shell | Use When |
|---------|-----------|-------|----------|
| **Unix** | `skills/index-knowledge-unix/` | bash, find, awk, wc, sed | macOS, Linux, WSL |
| **PowerShell** | `skills/index-knowledge-pwsh/` | PowerShell (`Get-ChildItem`, `ForEach-Object`, etc.) | Windows (native PowerShell) |

Both variants share the same workflow phases, scoring matrix, and output format. The only difference is the structural analysis commands in Phase 1 — Unix uses POSIX tools, Pwsh uses PowerShell cmdlets.

### Quick Comparison

| Aspect | Unix | PowerShell |
|--------|------|------------|
| File counting | `find . -type f \| wc -l` | `(Get-ChildItem -Recurse -File).Count` |
| Line counting | `wc -l` pipeline | `Measure-Object -Line` |
| Directory depth | `find \| awk -F/` | `Split-Path` + `.Count` |
| Code concentration | `find \| sed \| sort \| uniq` | `Group-Object \| Sort-Object` |
| Pattern filtering | `grep -r`, `find -name` | `Where-Object { $_.FullName -notmatch }` |

---

## Scoring Criteria

Directories are scored to determine which ones warrant their own `AGENTS.md`. The scoring matrix is:

| Factor | Weight | High Threshold | Source |
|--------|--------|----------------|--------|
| File count | 3× | >20 | shell |
| Subdir count | 2× | >5 | shell |
| Code ratio | 2× | >70% | shell |
| Unique patterns | 1× | Has own config | explore agents |
| Module boundary | 2× | Has `index.ts`/`__init__.py` | shell |
| Symbol density | 2× | >30 symbols | LSP |
| Export count | 2× | >10 exports | LSP |
| Reference centrality | 3× | >20 refs | LSP |

### Decision Rules

| Score | Action |
|-------|--------|
| Root (`.`) | **Always create** |
| >15 | Create `AGENTS.md` |
| 8–15 | Create if distinct domain |
| <8 | Skip (parent covers) |

---

## Usage

The skill is triggered by requests to "index", "document", "map", or "generate knowledge" for a codebase, or when onboarding onto a new project.

### Flags

```text
--create-new   # Read existing → remove all → regenerate from scratch
--max-depth=2  # Limit directory depth (default: 5)
```

Default mode is **update** — modify existing `AGENTS.md` files and create new ones where warranted.

---

## Installation

### Using `npx skills add` (Recommended)

This repo is compatible with the [`skills`](https://npm.im/skills) CLI — the open agent skills ecosystem. Install a specific variant:

```bash
# Install the Unix variant
npx skills add mynameistito/index-knowledge --skill index-knowledge-unix

# Install the PowerShell variant
npx skills add mynameistito/index-knowledge --skill index-knowledge-pwsh

# Install both variants
npx skills add mynameistito/index-knowledge --skill index-knowledge-unix --skill index-knowledge-pwsh
```

#### Scope Options

```bash
# Project-level (default) — committed with your project, shared with your team
npx skills add mynameistito/index-knowledge --skill index-knowledge-unix

# Global — available across all your projects
npx skills add mynameistito/index-knowledge --skill index-knowledge-unix -g

# Target a specific agent
npx skills add mynameistito/index-knowledge --skill index-knowledge-unix -a opencode
npx skills add mynameistito/index-knowledge --skill index-knowledge-pwsh -a claude-code
```

#### List Available Skills First

```bash
# See what's available in this repo before installing
npx skills add mynameistito/index-knowledge --list
```

#### Non-Interactive (CI/CD)

```bash
npx skills add mynameistito/index-knowledge --skill index-knowledge-unix -g -a opencode -y
```

### Other Methods

You can also install directly via the `add-skill` tool by Vercel Labs:

```bash
npx add-skill mynameistito/index-knowledge --skill index-knowledge-unix
```

Or manually copy the desired variant's folder into your agent's skills directory:

| Agent | Project Location | Global Location |
|-------|-----------------|-----------------|
| OpenCode | `./.opencode/skill/` | `~/.config/opencode/skill/` |
| Claude Code | `./.claude/skills/` | `~/.claude/skills/` |
| Codex | `./.codex/skills/` | `~/.codex/skills/` |
| Cursor | `./.cursor/skills/` | `~/.cursor/skills/` |

---

## How It Works

### Phase 1: Discovery + Analysis (Concurrent)

The skill launches parallel explore agents for project structure, entry points, conventions, anti-patterns, build/CI, and test patterns. Meanwhile, the main session runs shell structural analysis and reads any existing `AGENTS.md` files.

For larger projects, additional explore agents spawn dynamically based on file count, line count, directory depth, and language diversity.

### Phase 2: Scoring & Location Decision

Each directory is scored using the matrix above. High-scoring directories get their own `AGENTS.md`; low-scoring ones are covered by their parent.

### Phase 3: Generation

Root `AGENTS.md` gets the full treatment. Subdirectory files are generated in parallel by general agents, each scoped to 30–80 lines with no parent content duplication.

### Phase 4: Review & Deduplicate

Every generated file is reviewed to remove generic advice, parent duplicates, and verbosity violations.

---

## Dynamic Agent Spawning

The skill adapts to project scale by spawning additional explore agents:

| Factor | Threshold | Additional Agents |
|--------|-----------|-------------------|
| Total files | >100 | +1 per 100 files |
| Total lines | >10k | +1 per 10k lines |
| Directory depth | ≥4 | +2 for deep exploration |
| Large files (>500 lines) | >10 files | +1 for complexity hotspots |
| Monorepo | detected | +1 per package/workspace |
| Multiple languages | >1 | +1 per language |

---

## Anti-Patterns

The skill explicitly avoids these pitfalls:

- **Static agent count** — Always varies agents based on project size/depth
- **Sequential execution** — Always uses parallel Task calls in one message
- **Ignoring existing** — Always reads existing files first, even with `--create-new`
- **Over-documenting** — Not every directory needs an `AGENTS.md`
- **Redundancy** — Child files never repeat parent content
- **Generic content** — Removes anything that applies to *all* projects
- **Verbose style** — Telegraphic or die

---

## Project Structure

```
index-knowledge/
├── README.md                              # This file
├── LICENSE                                # MIT
└── skills/
    ├── index-knowledge-pwsh/
    │   ├── SKILL.md                       # Skill definition (frontmatter + full workflow)
    │   ├── metadata.json                  # Version, organization, abstract
    │   └── README.md                       # Variant-specific docs
    └── index-knowledge-unix/
        ├── SKILL.md                       # Skill definition (frontmatter + full workflow)
        ├── metadata.json                  # Version, organization, abstract
        └── README.md                       # Variant-specific docs
```

## License

MIT — see [LICENSE](./LICENSE).
This agent SKILL is a fork from dmmulroy's repo, but refactored to be PowerShell native as well.

