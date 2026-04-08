# Index Knowledge (Unix)

Generate a hierarchical AGENTS.md knowledge base for any codebase. Produces concise, machine-readable orientation docs scored by complexity and domain distinctness. Uses Unix shell tooling (bash, find, awk, wc) for structural analysis.

## Structure

- `SKILL.md` - Skill definition with frontmatter metadata and full workflow instructions
- `metadata.json` - Document metadata (version, organization, abstract)
- `AGENTS.md` - Compiled output (generated at runtime, placed in target projects)

## What It Does

1. **Discovery** - Scans project structure in parallel using explore agents + Unix shell analysis + LSP symbols
2. **Scoring** - Rates each directory by file count, symbol density, module boundaries, and export centrality
3. **Generation** - Produces root + subdirectory AGENTS.md files, each under 150 lines
4. **Review** - Deduplicates, trims, and validates output against quality gates

## Usage

```
# Default: update mode (modify existing + create new where warranted)
index-knowledge

# Regenerate from scratch
index-knowledge --create-new

# Limit directory depth
index-knowledge --max-depth=2
```

## Scoring Criteria

| Factor | Weight | High Threshold |
|--------|--------|----------------|
| File count | 3x | >20 |
| Subdir count | 2x | >5 |
| Code ratio | 2x | >70% |
| Unique patterns | 1x | Has own config |
| Module boundary | 2x | Has index.ts/__init__.py |
| Symbol density | 2x | >30 symbols |
| Export count | 2x | >10 exports |
| Reference centrality | 3x | >20 refs |

## AGENTS.md Output Format

Root files get the full treatment: `OVERVIEW`, `STRUCTURE`, `WHERE TO LOOK`, `CODE MAP`, `CONVENTIONS`, `ANTI-PATTERNS`, `COMMANDS`, `NOTES`.

Subdirectory files are leaner: `OVERVIEW`, `WHERE TO LOOK`, `CONVENTIONS`, `ANTI-PATTERNS` — never repeating parent content.

All files target 50-150 lines. No generic advice. No obvious info. Telegraphic style.

## Contributing

When modifying the skill:

1. Edit `SKILL.md` for workflow changes
2. Update `metadata.json` version for releases
3. Keep instructions parallel-first (multiple Task calls in one message)
4. Never add generic content — everything should be project-specific
5. Use Unix-compatible commands only (bash, find, awk, wc, sed, grep) — no PowerShell