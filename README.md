# skills

> Personal agent skills for Codex and other skill-aware coding agents.

This repository collects reusable skills under `skills/`. Each skill is a self-contained folder with its own `SKILL.md` and any supporting metadata, references, scripts, or assets it needs.

## Available Skills

| Skill | Purpose |
|-------|---------|
| `index-knowledge` | Generate hierarchical `AGENTS.md` knowledge bases for codebases. |
| `humanise` | Audit drafts for AI-sounding patterns and rewrite them to feel more natural while preserving meaning and tone. |

## Installation

### Using `npx skills add`

List the skills available from this repository:

```bash
npx skills add mynameistito/skills --list
```

Install a specific skill:

```bash
npx skills add mynameistito/skills/index-knowledge
npx skills add mynameistito/skills/humanise
```

Install globally:

```bash
npx skills add mynameistito/skills/humanise -g
```

Target a specific agent:

```bash
npx skills add mynameistito/skills/humanise -a codex
npx skills add mynameistito/skills/index-knowledge -a claude-code
```

### Manual Installation

Copy the desired skill folder into your agent's skills directory:

| Agent | Project Location | Global Location |
|-------|------------------|-----------------|
| OpenCode | `./.opencode/skill/` | `~/.config/opencode/skill/` |
| Claude Code | `./.claude/skills/` | `~/.claude/skills/` |
| Codex | `./.codex/skills/` | `~/.codex/skills/` |
| Cursor | `./.cursor/skills/` | `~/.cursor/skills/` |

## Skill Notes

### `index-knowledge`

Generates concise, hierarchical `AGENTS.md` files for a codebase. It scans project structure, scores directories by complexity and domain distinctness, writes a root knowledge file, and creates targeted subdirectory docs where they are useful.

The original version was found in [@dmmulroy](https://github.com/dmmulroy/)'s [.dotfiles](https://github.com/dmmulroy/.dotfiles) repository. This fork extends it with an OS-agnostic workflow that uses agent-native tools instead of platform-specific shell commands.

See `skills/index-knowledge/README.md` and `skills/index-knowledge/SKILL.md` for the full workflow.

### `humanise`

Reviews user-provided drafts for AI-sounding writing patterns, using Wikipedia's [Signs of AI writing](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing) as a heuristic reference. It rewrites text to sound more natural, personal, and context-aware while preserving the original meaning and tone, then explains the main changes.

See `skills/humanise/SKILL.md` for the workflow and `skills/humanise/references/signs-of-ai-writing.md` for the compact reference.

## Repository Layout

```text
skills/
├── LICENSE
├── README.md
└── skills/
    ├── humanise/
    │   ├── SKILL.md
    │   ├── agents/openai.yaml
    │   ├── metadata.json
    │   └── references/signs-of-ai-writing.md
    └── index-knowledge/
        ├── SKILL.md
        ├── README.md
        └── metadata.json
```

## License

MIT, see [LICENSE](./LICENSE).
