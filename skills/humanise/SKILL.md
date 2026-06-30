---
name: humanise
description: Audit and improve drafts that may sound AI-generated. Use when the user asks to humanize, humanise, de-AI, make writing sound natural, remove AI-sounding patterns, audit a message or draft for AI tells, preserve tone while rewriting, or rate how human a piece of writing feels.
license: MIT
metadata:
  author: mynameistito
  version: "1.0.0"
---

# Humanise

Audit any user-provided text for AI-sounding patterns, then rewrite it so it feels more natural, personal, and context-aware while preserving the user's meaning, intent, and baseline tone.

## Core Rule

Treat AI-writing signs as clues, not proof. Do not claim text is AI-written. Say "AI-sounding", "machine-like", "generic", or "polished in a way that may read as synthetic" when needed.

Before auditing, read `references/signs-of-ai-writing.md` if it has not already been loaded in the current turn.

## Workflow

1. Identify the text's purpose, audience, and tone from context.
2. Scan for AI-sounding patterns:
   - Generic significance, broad claims, or inflated stakes.
   - Canned transitions, symmetrical phrasing, and repeated sentence shapes.
   - Promotional, over-polished, or adjective-heavy language.
   - Vague attribution, unsupported claims, and bland consensus language.
   - Formulaic conclusions about "challenges", "future prospects", or "broader implications".
   - Markdown-like structure, title case headers, overuse of bolding, tables, or em dashes when the format does not call for them.
   - Placeholder language, knowledge-cutoff disclaimers, or meta-commentary aimed at the user.
3. Rewrite with human texture:
   - Keep the user's actual point, stance, and level of formality.
   - Prefer specific nouns and concrete verbs over abstract summary.
   - Vary sentence length and rhythm.
   - Use contractions where they fit the voice.
   - Remove stock phrases and tidy symmetry.
   - Let a little specificity, preference, uncertainty, or directness remain when it suits the draft.
   - Keep imperfections that feel intentional and useful; do not over-smooth.
4. Explain the main changes in 2-4 short bullets.
5. Optionally rate both versions if the user asks for ratings or if ratings would be useful:
   - `Original human feel: N/10`
   - `Revised human feel: N/10`

## Output Format

Default to this compact format:

```markdown
**Audit**
- ...

**Rewrite**
...

**What changed**
- ...

**Human feel**
Original: N/10
Revised: N/10
```

Omit the rating section when the user only wants a quick rewrite. For tiny drafts, collapse the audit and explanation into one or two sentences.

## Editing Guidance

- Preserve named facts, commitments, dates, claims, quotes, and technical meaning.
- Preserve the requested tone: professional, warm, casual, blunt, academic, persuasive, apologetic, etc.
- Do not add personal anecdotes unless the user gave enough context to support them.
- Do not make the text sloppy just to make it "human".
- Avoid overusing contractions, fragments, jokes, idioms, or emotional language.
- If the original has serious factual, legal, medical, or financial claims, flag unsupported assertions instead of making them sound more confident.
- If the user asks for a direct replacement, provide the rewritten text first and keep the audit very short.
