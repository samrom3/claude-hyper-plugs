# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Hyperteam is a Claude Code plugin that provides two skills (`/prd` and `/hyperteam`) for AI-augmented development. It interviews the user to build a PRD, then runs an autonomous specialist agent team with back-pressure gates to implement it.

This is **not** a traditional software project with build/test/lint commands. It is a collection of Markdown-based agent definitions and skill specifications that form a Claude Code plugin.

## Verification

Pre-commit hooks handle formatting and linting:

```bash
pre-commit run --all-files    # trailing whitespace, EOF fixer, YAML check, merge conflict check, mdformat
```

mdformat (with GFM, footnote, config, ruff, and toc plugins) runs on all Markdown files **except** those under `skills/` and `agents/` (excluded because agent/skill prompts use intentional formatting).

## Architecture

### Plugin entry point

`.claude-plugin/plugin.json` — declares the plugin metadata. Claude Code discovers skills from `skills/` and agents from `agents/`.

### Two-step workflow

1. **`/prd`** (`skills/prd/SKILL.md`) — Multi-phase PRD generator. Interviews the user (3 phases of refinement with conflict detection), outputs `plans/<branch>-prd.md`. Creates a `feat-<slug>` branch and sets `CLAUDE_CODE_TASK_LIST_ID` in `.claude/settings.local.json`.
1. **`/hyperteam`** (`skills/hyperteam/SKILL.md`) — Parses the PRD into a task DAG, creates a specialist agent team via `TeamCreate`, seeds native tasks, and monitors until the back-pressure gate passes. Supports mid-run resume via `plans/<branch>-team-state.json`.

### Agent roles

- **`hyperteam-lead`** — Orchestrator. Monitors the run, handles review failures, detects gate readiness. Does not implement code. Runs on Sonnet.
- **`hyperteam-reviewer`** — Reviews completed FEAT tasks against acceptance criteria, runs the 5-check gate. Read-only for source code. Runs on Sonnet.
- **`hyperteam-worker`** — Fallback implementer. Claims tasks via self-claim loop, follows TDD, runs verification before committing. Runs on Sonnet.
- **`hyperteam-techwriter`** — Claims DOC tasks, updates documentation. DOC tasks skip review (terminal state is `completed`). Runs on Sonnet.
- **Python pack** (`agents/packs/python/`) — `hyperteam-py-api-scaffolder` (creates stubs/ABCs) and `hyperteam-py-builder` (implements via TDD on scaffolds). The lead auto-activates these for Python tasks.

### State files (created at runtime under `plans/`)

- `plans/<branch>-team-state.json` — Authoritative task registry (status, blockers, review results, gate iterations). Enables resume.
- `plans/<branch>-progress.txt` — Append-only audit log with timestamps.
- `plans/<branch>-prd.md` — The generated PRD document.

### Key conventions

- Agents self-claim tasks from the native task list; the lead does not dispatch workers manually.
- Tasks use YAML front-matter in their description (`id`, `type`, `role_hint`, `blocked_by`).
- Commit messages follow `[Story-ID] - [Story Title]` format.
- All implementing agents must read `CLAUDE.md` of the target project and follow its conventions.
- The scaffold-first pattern is used: first story creates typed stubs with `NotImplementedError` bodies, subsequent stories implement via TDD.

### Language packs

Custom language packs go in the target project's `.claude/agents/` directory using the naming convention `hyperteam-<lang>-<role>.md`. The phase-1 role-hint system auto-routes tasks to matching specialists.
