---
name: hyperwork-tech-writing
description: Technical writing — READMEs, changelogs, ADRs, docstrings, and user-facing docs. Match repo style, compress prose, preserve structure.
user-invocable: false
---

# Tech Writing

**Use:** Doc tasks — READMEs, CHANGELOG entries, ADRs, inline docstrings, migration guides.

## Approach

1. Read existing docs in same dir/format first — match voice, structure, heading depth.
2. Identify audience: user-facing (full sentences, no jargon) vs. agent-consumed (ultra caveman).
3. Write minimum content that satisfies the task. No padding, no hedging.
4. Verify all code examples run / match current API.

## Compression Rules (agent-consumed files)

- Drop articles (a/an/the), conjunctions, hedging phrases.
- Arrows for causality: X → Y.
- One-line reminders instead of paragraphs for quality criteria.
- Preserve: numbered steps, code blocks, YAML front-matter, table structure, quoted strings.

## Format Rules (user-facing files)

- Lite compression: drop filler/hedging, keep full sentences.
- Tables for comparisons. Code blocks for all commands.
- Keep a Changelog format for CHANGELOG.md: `## [version]`, `### Added/Changed/Fixed/Removed`.

## Do / Don't

- DO: run `pre-commit run --all-files` — mdformat enforces Markdown style on non-agent files.
- DO: verify links and file paths exist before writing them.
- DON'T: add a comment that just restates what the code does.
- DON'T: document hypothetical future behaviour — only current behaviour.
- DON'T: leave placeholder text (TODO, TBD, fill in) — write the content or leave the section out.
