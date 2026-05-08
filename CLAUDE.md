# CLAUDE.md

Claude Code guidance for this repo.

## What This Is

Plugin marketplace. Each plugin: self-contained dir with `.claude-plugin/plugin.json`, agents, skills, docs.

Not a traditional project. No build/test/lint. Markdown agent definitions + skill specs only.

## Repository Structure

```
claude-hyper-plugs/
├── CLAUDE.md                  # This file (repo-level guidance)
├── README.md                  # Marketplace index and installation guide
├── .pre-commit-config.yaml    # Shared formatting/linting hooks
└── <plugin-name>/             # One directory per plugin
    ├── .claude-plugin/
    │   └── plugin.json        # Plugin metadata (name, version, description, author)
    ├── agents/                # Agent role definitions (Markdown with YAML front-matter)
    ├── skills/                # Skill definitions (SKILL.md + references/)
    └── README.md              # Plugin-specific documentation
```

## Current Plugins

| Plugin       | Directory     | Description                                                                                        |
| ------------ | ------------- | -------------------------------------------------------------------------------------------------- |
| `hyperloop`  | `hyperloop/`  | Autonomous specialist agent team with PRD-driven planning and gates                                |
| `adr-wizard` | `adr-wizard/` | ADR lifecycle management: create, supersede, deprecate, and validate Architecture Decision Records |

## Verification

Pre-commit: formatting, linting, plugin validation.

```bash
pre-commit run --all-files    # trailing whitespace, EOF fixer, YAML check, merge conflict check, mdformat, plugin validate
```

- mdformat: all Markdown **except** `skills/` and `agents/` (intentional formatting excluded)
- `claude plugin validate .`: validates marketplace manifest + all plugin manifests/frontmatter

## Adding a New Plugin

1. Create `<plugin-name>/` at repo root
1. Add `.claude-plugin/plugin.json`
1. Add `agents/` and/or `skills/` dirs
1. Add `README.md`
1. Update "Current Plugins" table + root `README.md`

## Plugin Anatomy

- `.claude-plugin/plugin.json` — name, version, description, author, license
- `agents/<name>.md` — YAML front-matter: `name`, `description`, `model`, `permissionMode`
- `skills/<name>/SKILL.md` — YAML front-matter: `name`, `description`, `user-invocable`. Refs → `references/`

## Plugin Conventions

### Agent directory — always flat

Plugin agents **must** live directly in `agents/` — no subdirectories. Nested agents are invisible to discovery.

- Correct: `hyperloop/agents/hyperteam-worker.md`
- Wrong: `hyperloop/agents/packs/hyperteam-worker.md`

> User-added agents (in `.claude/agents/`) not subject to this rule.

### Skill directory — always flat, prefixed

Plugin skills **must** live directly in `skills/<name>/SKILL.md` — no subdirectories. Nested skills not loaded by Claude Code.
Prefix skill names with the plugin identifier to avoid collisions across plugins.

- Correct: `hyperloop/skills/hyperwork-tdd/SKILL.md`
- Wrong: `hyperloop/skills/worker-skills/tdd/SKILL.md`

### Versioning — semantic versioning

| Change type                                        | Version bump | Example           |
| -------------------------------------------------- | ------------ | ----------------- |
| Breaking change (removes or renames agents/skills) | **major**    | `1.0.0` → `2.0.0` |
| New feature (adds agents, skills, or capabilities) | **minor**    | `1.0.0` → `1.1.0` |
| Bug fix, documentation, refactor (no API change)   | **patch**    | `1.0.0` → `1.0.1` |

Bump `plugin.json` on every publish. CC caches manifests — no bump → users miss update.

### CHANGELOG — required for every version bump

- Keep a Changelog format. File inside plugin dir (independently versioned).
- Use `[Unreleased]` for WIP. Entries: `### Added/Changed/Fixed/Removed`.

### README sync — required when skill interfaces change

Update plugin `README.md` when: arg format/`argument-hint` changes, skills added/removed, user-visible behavior changes. README examples must match actual invocation syntax.

## Skill and Reference File Compression

All skill/reference files must be compressed before commit. Level depends on file type:

| File type                                         | Level                                                   |
| ------------------------------------------------- | ------------------------------------------------------- |
| Agent-consumed `SKILL.md`                         | Ultra caveman                                           |
| Agent-consumed reference files                    | Ultra caveman                                           |
| `CLAUDE.md` (this file)                           | Ultra caveman                                           |
| Machine-parseable contract/format spec tables     | Preserve structure; compress surrounding prose only     |
| User-facing templates (copied into user projects) | Lite caveman (drop filler/hedging; keep full sentences) |
| Template files ≤20 lines                          | Near-zero-change — leave untouched                      |

**Ultra — drop:** articles (a/an/the), conjunctions, hedging, pleasantries, multi-sentence quality-criteria paragraphs → one-line reminders (e.g., `*Context: describe problem forces, not solution.*`).

**Arrows for causality:** X → Y.

**Preserve always:** numbered step sequence, code blocks, YAML front-matter, table structure in spec/contract files, quoted error strings, conditional logic branches, AC lists.

**Review reminder:** compress before committing new/edited skill files. PRs adding skill/reference content → include before/after line counts in PR body.
