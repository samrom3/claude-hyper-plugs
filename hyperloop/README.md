# Hyperloop

AI-augmented development for engineers who stay in the loop.

Hyperloop interviews you to build a session-spec (with requirement analysis and conflict resolution), then
runs an autonomous specialist agent team with back-pressure gates to implement it — while you
review, guide, and approve at every milestone.

## Prerequisites

- **Claude Code** — see the [official installation guide](https://docs.anthropic.com/en/docs/claude-code/setup).
- **Agent Teams** must be enabled. This is an experimental feature — see [Claude Code: Enable Agent Teams](https://code.claude.com/docs/en/agent-teams#enable-agent-teams) for details.
- **`gh` CLI** installed and authenticated (for PR creation).
- A **`CLAUDE.md`** at your project root that documents:
  - Your project's verification command (lint + format + tests), e.g. `uv run pre-commit run`,
    `npm run lint && npm test`, `cargo clippy && cargo test`.
  - Source directory layout and conventions
  - If you don't have one, then use `claude --init` to create one.

## Installation

Hyperloop is installed via the [hyper-plugs marketplace](../README.md):

```
/plugin marketplace add samrom3/claude-hyper-plugs
/plugin install hyperloop@hyper-plugs
```

See the [plugin marketplaces documentation](https://code.claude.com/docs/en/plugin-marketplaces) for details.

## Usage

Hyperloop is a two-step process. First you build a session-spec through a structured interview, then the agent team executes it autonomously against a back-pressure gate:

```
                    ┌─────────────────────────────────────────────────────────┐
  Step 1            │                                                         │
  (/session-spec)   │   You ──► Interview ──► Conflict ──► plans/<branch>     │
                    │            (1 pass)      analysis    -session-spec.md   │
                    └──────────────────────────────┬──────────────────────────┘
                                                   │ session-spec
                    ┌──────────────────────────────▼──────────────────────────┐
  Step 2            │         Lead parses session-spec → task DAG             │
  (/hyperteam)      │                                                         │
                    │   ┌─────────┐   ┌─────────┐   ┌─────────┐              │
                    │   │Worker 1 │   │Worker 2 │   │Worker N │  . . .       │
                    │   └────┬────┘   └────┬────┘   └────┬────┘              │
                    │        └────────────┬┘              │                   │
                    │                     ▼                │                   │
                    │               ┌──────────┐          │                   │
                    │               │ Reviewer │◄─────────┘                   │
                    │               │  [GATE]  │                              │
                    │               └────┬─────┘                              │
                    │                    │ pass                                │
                    │                    ▼                                     │
                    │                   PR                                     │
                    └─────────────────────────────────────────────────────────┘
```

### 1. Generate a session-spec

```
/hyperloop:session-spec Add user authentication with OAuth2 support
```

or from a seedling document:

```
/hyperloop:session-spec plans/auth-seedling.md
```

or linked to a GitHub issue:

```
/hyperloop:session-spec https://github.com/owner/repo/issues/42 Add user authentication
```

The session-spec skill will:

- Interview you to gather requirements (single focused round)
- Cross-check against existing ADRs and codebase patterns
- Surface conflicts and ambiguities before any code is written
- Output `plans/<branch>-session-spec.md`

You can generate multiple session-specs upfront — they accumulate in `plans/` and can be executed in any order.

#### GitHub issue linking

If you include a GitHub issue URL anywhere in your `/session-spec` arguments, hyperloop will:

1. **Assign the issue** — runs `gh issue edit <N> --repo <owner>/<repo> --add-assignee @me`
   immediately, so the issue is claimed as soon as PRD work begins. If `gh` is unauthenticated or
   the assignment fails for any reason, a warning is printed but PRD creation continues.

1. **Record the reference in the spec** — writes a metadata table immediately after the spec's H1
   heading:

   ```markdown
   # My Feature

   | Field        | Value              |
   | ------------ | ------------------ |
   | Source Issue | owner/repo#42      |

   ## Goal
   ```

1. **Propagate to `team-state.json`** — Phase 1 reads the metadata table and stores
   `metadata.source_issues` in `team-state.json` as the authoritative runtime value.

1. **Auto-close via PR** — when the gate passes and a PR is created, Phase 4 appends
   `Closes #N` (same-repo) or `Closes https://github.com/owner/repo/issues/N` (cross-repo) to
   the PR body, so the issue closes automatically on merge.

### 2. Run the team

> BEST-PRACTICE: Run `/clear` first to avoid context drift and costs of auto compaction.

```
/hyperloop:hyperteam
```

At startup, `/hyperloop:hyperteam` scans `plans/` for `*-session-spec.md` files and prompts you to select one:

- If only one incomplete session-spec exists, it confirms and auto-selects it.
- If multiple incomplete session-specs exist, it presents a numbered list for you to choose from.
- In-progress session-specs (those with an existing `team-state.json`) are flagged with a warning so you don't accidentally start a duplicate session.

After session-spec selection, hyperloop will:

- Parse the PRD into a dependency-ordered task DAG
- Show you the plan and wait for approval
- Create a three-agent team (lead + N workers + reviewer; N inferred from parallel-eligible task count)
- Execute all tasks with TDD, code review, and a back-pressure gate
- Offer to create a PR when everything passes

### Resuming interrupted runs

If a session ends mid-run (quota exhaustion, network drop, etc.), just run `/hyperloop:hyperteam` again.
It detects the existing `team-state.json`, shows you what's done and what's left, and picks up
where it stopped.

### Concurrent sessions

Each `/hyperloop:hyperteam` session creates its own agent team via `TeamCreate`, which
automatically scopes the native task list by team name. This means you can run multiple sessions
in parallel — open separate Claude Code sessions, then run `/hyperloop:hyperteam` in each, and select a
different session-spec in each session. Sessions are fully isolated because each team has its own task list
at `~/.claude/tasks/{team-name}/`.

## Architecture

### Skills

| Skill                     | Purpose                                                                             |
| ------------------------- | ----------------------------------------------------------------------------------- |
| `/hyperloop:session-spec` | Single-pass session-spec generator with conflict detection and requirement analysis |
| `/hyperloop:hyperteam`    | Converts a session-spec into an autonomous agent team with back-pressure gates      |

### Agents

| Agent                | Role                                                                                                           |
| -------------------- | -------------------------------------------------------------------------------------------------------------- |
| `hyperteam-lead`     | Orchestrator — monitors the run, handles failures, detects gate readiness; consult-arbiter for worker blockers |
| `hyperteam-reviewer` | Reviews completed work against acceptance criteria; emits structured PASS/FAIL; runs the 5-check gate          |
| `hyperteam-worker`   | Primary executor — claims all tasks, loads assigned skills at claim time, implements, commits                  |

### Worker Skills Library

Workers load domain skills at task-claim time via the `Skill` tool. Skills are assigned per task in the session-spec and stored in task YAML front-matter (`skills:` field). Skill files live in `hyperloop/skills/worker-skills/`.

| Skill            | Purpose                                                            |
| ---------------- | ------------------------------------------------------------------ |
| `tdd-python`     | Python implementation: pytest-first, type hints, uv/venv tooling   |
| `tdd-typescript` | TypeScript implementation: tsc type check, vitest/ts-jest patterns |
| `api-scaffold`   | API endpoint stubs, schema generation, contract-first design       |
| `tech-writing`   | Docs, README, changelog, ADR, user-facing writing                  |
| `tdd-generic`    | Fallback for untyped or mixed tasks not matching a specific skill  |

### State management

- **`plans/<branch>-team-state.json`** — Authoritative task registry. Tracks status, blockers,
  review results, and gate iterations. Enables mid-run resume.
- **`plans/<branch>-progress.txt`** — Append-only audit log with timestamps.
- **Native task list** — Live coordination bus for agent self-claiming.

## Customization

### Verification commands

Hyperloop agents read `CLAUDE.md` to find your project's lint/test command. Ensure your
`CLAUDE.md` has a clear "Tooling" section, for example:

````markdown
## Tooling

```bash
npm run lint          # lint + format
npm test              # run tests
npm run check         # both (use this as the verification command)
`` `
````

### Overriding the session-spec template

Place your own `example-session-spec.md` at `.claude/skills/session-spec/references/example-session-spec.md` in your
project. Project-level files take precedence over plugin files.

### Project-specific agents

Add agents to your project's `.claude/agents/` directory. They supplement the plugin's agents
and take precedence when names overlap.

## Inspiration & prior art

**Inspired by [hyperworker](https://github.com/joseph-ravenwolfe/hyperworker)** by Joseph Ravenwolfe.

**Key architectural differences in hyperloop:**

|                           | hyperworker                                    | hyperloop                                                                                                            |
| ------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Language support**      | Pick the language-specific version of the repo | Single install; worker-skills library covers Python, TypeScript, API scaffold, docs — works in mixed-stack repos     |
| **Requirement gathering** | Prompt-driven                                  | Structured user interview with explicit requirement analysis and conflict deconfliction before any code is written   |
| **Back-pressure gate**    | Intended but consistently absent in most forks | First-class GATE tasks with a dedicated reviewer agent; the lead blocks new work until the gate clears               |
| **Re-entrant execution**  | Single run                                     | Designed for mid-run quota exhaustion: resume picks up exactly where the last session left off via `team-state.json` |

## License

MIT
