# index-knowledge

> Generate hierarchical AGENTS.md knowledge bases for any codebase — concise enough for LLM context windows, specific enough to be useful. Fully OS-agnostic: macOS, Linux, WSL, Windows PowerShell, Windows CMD.

Orignally found in [@dmmulroy](https://github.com/dmmulroy/)'s [.dotfiles](https://github.com/dmmulroy/.dotfiles) repository. This repo forks and extends the skill with a unified OS-agnostic variant that uses agent-native tools instead of platform-specific shell commands.

---

## What It Does

Most codebases lack machine-readable orientation — no quick map of where things live, what conventions matter, or which directories are complexity hotspots. `index-knowledge` fixes that by producing a hierarchy of concise `AGENTS.md` files:

1. **Discovery** — Scans the project tree in parallel using explore agents, agent-native structural analysis (glob, grep, read), and LSP symbols
2. **Scoring** — Rates each directory by file count, symbol density, module boundaries, and export centrality
3. **Generation** — Produces a root `AGENTS.md` (full treatment) and targeted subdirectory docs (leaner)
4. **Review** — Deduplicates, trims, and validates output against quality gates

Every generated file stays under 150 lines. No generic advice. No obvious info. Telegraphic style.

### Root AGENTS.md (Full Treatment)

Root files include: `OVERVIEW`, `STRUCTURE`, `WHERE TO LOOK`, `CODE MAP`, `CONVENTIONS`, `ANTI-PATTERNS`, `COMMANDS`, `NOTES`.

### Subdirectory AGENTS.md (Lean)

Subdirectory files include: `OVERVIEW`, `WHERE TO LOOK`, `CONVENTIONS`, `ANTI-PATTERNS` — never repeating parent content.

---

## Platform Support

This skill is fully OS-agnostic. All structural analysis uses **agent-native tools** (glob, grep, read) that work identically on every platform. No bash or PowerShell required.

| Platform | Supported |
|----------|-----------|
| macOS | Yes |
| Linux | Yes |
| WSL | Yes |
| Windows PowerShell | Yes |
| Windows CMD | Yes |

---

## Scoring Criteria

Directories are scored to determine which ones warrant their own `AGENTS.md`. The scoring matrix is:

| Factor | Weight | High Threshold | Source |
|--------|--------|----------------|--------|
| File count | 3× | >20 | glob |
| Subdir count | 2× | >5 | read |
| Code ratio | 2× | >70% | glob |
| Unique patterns | 1× | Has own config | explore |
| Module boundary | 2× | Has index.ts/__init__.py | glob |
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

This repo is compatible with the [`skills`](https://npm.im/skills) CLI — the open agent skills ecosystem.

```bash
npx skills add mynameistito/index-knowledge
```

#### Scope Options

```bash
# Project-level (default) — committed with your project, shared with your team
npx skills add mynameistito/index-knowledge

# Global — available across all your projects
npx skills add mynameistito/index-knowledge -g

# Target a specific agent
npx skills add mynameistito/index-knowledge -a opencode
npx skills add mynameistito/index-knowledge -a claude-code
```

#### List Available Skills First

```bash
npx skills add mynameistito/index-knowledge --list
```

#### Non-Interactive (CI/CD)

```bash
npx skills add mynameistito/index-knowledge -g -a opencode -y
```

### Other Methods

You can also install directly via the `add-skill` tool by Vercel Labs:

```bash
npx add-skill mynameistito/index-knowledge
```

Or manually copy the skill folder into your agent's skills directory:

| Agent | Project Location | Global Location |
|-------|-----------------|-----------------|
| OpenCode | `./.opencode/skill/` | `~/.config/opencode/skill/` |
| Claude Code | `./.claude/skills/` | `~/.claude/skills/` |
| Codex | `./.codex/skills/` | `~/.codex/skills/` |
| Cursor | `./.cursor/skills/` | `~/.cursor/skills/` |

---

## How It Works

### Phase 1: Discovery + Analysis (Concurrent)

The skill launches parallel explore agents for project structure, entry points, conventions, anti-patterns, build/CI, and test patterns. Meanwhile, the main session runs agent-native structural analysis (glob, grep, read) and reads any existing `AGENTS.md` files.

For larger projects, additional explore agents spawn dynamically based on file count, line count, directory depth, and language diversity.

### Phase 2: Scoring & Location Decision

Each directory is scored using the matrix above. High-scoring directories get their own `AGENTS.md`; low-scoring ones are covered by their parent.

### Phase 3: Generation

Root `AGENTS.md` gets the full treatment. Subdirectory files are generated in parallel by general agents, each scoped to 50–150 lines with no parent content duplication.

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
- **Platform-specific commands** — Always uses agent-native tools (glob, grep, read), never assumes bash or PowerShell

---

## Project Structure

```
index-knowledge/
├── README.md                              # This file
├── LICENSE                                # MIT
└── skills/
    └── index-knowledge/
        ├── SKILL.md                       # Skill definition (frontmatter + full workflow)
        ├── metadata.json                  # Version, organization, abstract
        └── README.md                      # Skill docs
```

## License

MIT — see [LICENSE](./LICENSE).
Originally forked from dmmulroy's repo, refactored to be fully OS-agnostic using agent-native tools.
