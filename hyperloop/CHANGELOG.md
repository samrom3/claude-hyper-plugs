# Changelog

All notable changes to the hyperloop plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [3.0.0] - 2026-05-08

### Added

- Five loadable worker skills at `skills/hyperwork-*/`: `hyperwork-tdd`, `hyperwork-python`,
  `hyperwork-typescript`, `hyperwork-api-scaffold`, `hyperwork-tech-writing`. Workers load
  assigned skills at task-claim time via the `Skill` tool.
- `skills: none` sentinel in session-spec annotations — explicit signal that a step requires
  no skill loading (config, env setup, non-code steps).

### Changed

- `hyperteam-worker` promoted to primary executor — claims all FEAT and DOC tasks by `type`
  field; loads skills from `skills:` front-matter before work.
- `hyperteam-lead` extended with consult-arbiter protocol — receives worker blockers/questions
  via `SendMessage`, decides to unblock or escalate upstream. Workers never contact user directly.
- `hyperteam-reviewer` updated to emit structured PASS/FAIL output — `result`, `task_id`,
  `findings` (FEAT review) and per-check status (GATE); no prose narratives.
- `/hyperteam` Phase 2 team composition — N workers inferred from parallel-eligible task count
  at kickoff (clamped 1–4).
- `session-spec` skill assigns `skills:` array per step using five match rules; task routing
  driven by `type` field — `role_hint` field removed entirely.

### Removed

- `hyperteam-py-builder` agent
- `hyperteam-py-api-scaffolder` agent
- `hyperteam-techwriter` agent

## [2.0.0] - 2026-05-07

### Added

- `session-spec` skill — single-pass replacement for `prd`: one interview round, step→verify
  output format, adversarial conflict detection preserved.
- Gate discretion in `/hyperteam`: recommends skipping back-pressure gate for small specs
  (≤3 steps, no cross-step deps, human in verification loop). User can override either way.
- ADR-003 supersedes ADR-002 — updates contract writer from `prd` to `session-spec` skill and
  first section heading from `## 1. Introduction/Overview` to `## Goal`.

### Changed

- Output files renamed: `plans/<branch>-session-spec.md` (was `-prd.md`).
- `/hyperteam` warns on legacy `-prd.md` files — will not process silently.
- All internal cross-references updated: ADR-001, ADR-002 (superseded), README, skill tables,
  reference files.

### Removed

- `prd` skill (breaking). Use `/session-spec` going forward.

## [1.3.1] - 2026-05-07

### Changed

- Ultra caveman compression applied to `prd/SKILL.md` (201→~130 lines) and
  `hyperteam/SKILL.md` (203→~130 lines) — articles/conjunctions/hedging dropped; arrows for
  causality; "Seedling philosophy" blockquote condensed to 1–2 lines; Critical Review Mandate
  multi-sentence rationale → 1–2 line imperatives; Phase notes → one-line callouts; all
  phase/step structure, `AskUserQuestion` call specs, `git` command sequences, and conditional
  logic branches preserved intact
- Ultra caveman compression applied to all six reference files:
  `phase-1-fresh-start.md`, `phase-1-resume.md`, `phase-4-completion.md` (numbered steps
  preserved in order); `gate-task-template.md` (gate check sequence Check 1–5 and
  failure-escalation logic preserved); `team-state-schema.md` (JSON examples remain valid,
  field names/status values/timestamps unchanged, field description cells compressed);
  `example-prd.md` (`> **[Guidance]**` blockquotes compressed, PRD body fully preserved)

## [1.2.0] - 2026-03-27

### Changed

- Gate Check 2 (ADR sync) now delegates to the `/adr-check` skill when adr-wizard is installed,
  falling back to manual directory scanning when it is not. The check no longer hardcodes a
  single `docs/adrs/` path — it reads `### ADR Locations` from `CLAUDE.md` or scans common
  fallback directories. Backwards-compatible: hyperteam works without adr-wizard installed.

## [1.1.0] - 2026-03-26

### Added

- PRD selection at `/hyperteam` startup — scans `plans/` for `*-prd.md` files, categorises them
  as unstarted or in-progress, and prompts the user to choose; supports creating several PRDs
  upfront and executing them in any order
- Concurrent session support — each `/hyperteam` session creates its own agent team via
  `TeamCreate`, which automatically scopes the native task list by team name; multiple sessions
  can run against different PRDs without interfering with each other.

### Changed

- Phase 0 of `/hyperteam` now scans `plans/` for PRDs instead of reading
  `CLAUDE_CODE_TASK_LIST_ID` from `.claude/settings.local.json`
- Phase 0 git branch step is now automatic: if the selected PRD's branch differs from the current
  branch, `/hyperteam` checks it out locally or creates it from `origin/main`
- `/prd` no longer writes `CLAUDE_CODE_TASK_LIST_ID` to `.claude/settings.local.json`; task list
  scoping is handled automatically by `TeamCreate` via team name.

## [1.0.1] - 2026-03-26

### Added

- This CHANGELOG

### Changed

- Flattened `agents/packs/python/` into `agents/` for automatic discovery by Claude Code

### Fixed

- Agent discovery for Python pack agents (`hyperteam-py-builder`, `hyperteam-py-api-scaffolder`) —
  previously invisible due to subdirectory nesting (Claude Code only scans the top-level `agents/` directory)

## [1.0.0] - 2026-03-23

### Added

- PRD generator skill (`/hyperloop:prd`) — multi-phase structured interview with requirement
  analysis and conflict deconfliction before any code is written
- Autonomous agent team skill (`/hyperloop:hyperteam`) — converts a PRD into a dependency-ordered
  task DAG and runs a specialist agent team with back-pressure gates
- Back-pressure gate (`hyperteam-reviewer`) — dedicated GATE task type; the lead blocks new work
  until the reviewer clears all acceptance criteria
- Re-entrant execution — `team-state.json` enables mid-run resume after quota exhaustion or
  network interruptions
- Python language pack — `hyperteam-py-api-scaffolder` (scaffold-first interface definitions) and
  `hyperteam-py-builder` (TDD business logic implementation)
- Core agents: `hyperteam-lead`, `hyperteam-reviewer`, `hyperteam-techwriter`, `hyperteam-worker`
