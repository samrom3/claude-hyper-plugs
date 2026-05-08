# ADR-001: Remove `CLAUDE_CODE_TASK_LIST_ID` from Persisted Config

**Status:** Accepted

**Date:** 2026-03-26

______________________________________________________________________

## Context

Hyperteam previously relied on the `/session-spec` skill to write `CLAUDE_CODE_TASK_LIST_ID` into
`.claude/settings.local.json` under `env`. This was intended to scope the native task list for
the hyperteam session.

However, this approach was based on a misunderstanding of how Agent Teams scope their task lists.
When `TeamCreate` is called with a `team_name`, Claude Code automatically:

1. Creates the task list directory at `~/.claude/tasks/{team-name}/`.
1. Sets `CLAUDE_CODE_TEAM_NAME` on all spawned teammates.
1. Scopes all `TaskCreate`, `TaskList`, and `TaskUpdate` calls to `~/.claude/tasks/{team-name}/`.

See: [Agent Teams architecture](https://code.claude.com/docs/en/agent-teams#architecture).

`CLAUDE_CODE_TASK_LIST_ID` is a **separate** mechanism for sharing a task list across non-team
sessions (see [env vars](https://code.claude.com/docs/en/env-vars)). It does not influence team
task list naming — the team name does.

Persisting `CLAUDE_CODE_TASK_LIST_ID` in `.claude/settings.local.json` was therefore unnecessary
for the team-based hyperteam workflow and introduced a false coupling between `/session-spec` and
`/hyperteam`.

______________________________________________________________________

## Decision

Remove `CLAUDE_CODE_TASK_LIST_ID` from the hyperteam workflow entirely:

- `/session-spec` no longer writes `CLAUDE_CODE_TASK_LIST_ID` to `.claude/settings.local.json`.
- `/hyperteam` no longer reads or exports `CLAUDE_CODE_TASK_LIST_ID`.
- Task list scoping is handled automatically by `TeamCreate` — hyperteam calls
  `TeamCreate` with `team_name: "<branch>"`, which creates the task list at
  `~/.claude/tasks/<branch>/`.

The `/session-spec` skill continues to create the `plans/<branch>` → `~/.claude/tasks/<branch>` symlink,
giving users local visibility into the task list from within the project directory.

______________________________________________________________________

## Consequences

**Positive:**

- **Concurrent runs work correctly.** Each `/hyperteam` session creates its own team via
  `TeamCreate` with a unique team name. Task lists are automatically isolated by team name —
  no shared config file to clobber.
- **No manual configuration required.** Users don't need to set any environment variables.
  `TeamCreate` handles scoping internally.
- **`/session-spec` is decoupled from `/hyperteam`.** Users can create several session-specs upfront and run
  `/hyperteam` against any of them in any order, in any session.
- **Correct mental model.** The architecture now reflects how Agent Teams actually work, rather
  than layering an unrelated env var mechanism on top.

**Neutral:**

- The `plans/<branch>` symlink (created by `/session-spec`) is still useful for local file visibility.
  Hyperteam verifies the symlink exists after spec selection and creates it if missing.

**Negative / Trade-offs:**

- None identified. The previous `CLAUDE_CODE_TASK_LIST_ID` usage was unnecessary for team
  workflows and its removal simplifies the architecture.
