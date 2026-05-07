---
name: hyperteam
description: "Reads a PRD, derives a task DAG, gets user approval, writes team-state.json, seeds the native task list, creates a specialist team, and monitors until the back-pressure gate passes. Replaces the /prd-tasks + /hyperworker two-step workflow."
user-invocable: true
disable-model-invocation: true
---

# Hyperteam

Converts PRD into autonomous agent team. Executes full task DAG, tracks state in `plans/<branch>-team-state.json`, coordinates via native task list, offers PR when gate passes.

______________________________________________________________________

## Phase 0: Pre-Flight

> **Prerequisites:** Requires Agent Teams feature + `gh` CLI. Set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Verify `gh` CLI installed and authenticated for PR creation.

Run checks in order. **Stop and surface each issue as encountered.**

### Step 1 — Scan `plans/` and select a PRD

1. List all files in `plans/` matching `*-prd.md`.
2. For each `plans/<name>-prd.md`, determine state:
   - No `plans/<name>-team-state.json` → **unstarted**.
   - `metadata.status = "running"` → **in-progress**.
   - `metadata.status = "complete"` → **complete**.
   - Any other value → **in-progress**.
3. Exclude **complete** PRDs from selection.
4. No incomplete PRDs → stop:
   > No incomplete PRDs found in `plans/`. Create a PRD first with `/prd`.
5. Build ordered selection list: unstarted first, then in-progress. Within each group, sort by file modification time (most recent first). Format each entry: `<n>. plans/<name>-prd.md`. Append for in-progress entries: `⚠ This PRD may have an in-flight hyperteam run. Ensure no other session is working on it before proceeding.`
6. **Single PRD:** Use `AskUserQuestion`:
   > Only one incomplete PRD found:
   >
   > `plans/<name>-prd.md` [warning if in-progress]
   >
   > Proceed with this PRD?

   User confirms → select it. Otherwise stop.
7. **Multiple PRDs:** Use `AskUserQuestion`:
   > Multiple PRDs found. Choose one to run:
   >
   > <numbered selection list>

   Wait for user's choice.
8. Derive `<branch>` from selected filename: strip `plans/` prefix and `-prd.md` suffix.
9. Derive `<slug>` from `<branch>` by stripping leading `feat-` prefix if present. If `<branch>` does not start with `feat-`, use `<branch>` as `<slug>` unchanged.

### Step 2 — Checkout git branch

1. Run `git branch --show-current`.
2. Result matches `<branch>` → proceed to Step 3.
3. Mismatch:
   a. Run `git branch --list <branch>`.
   b. **Branch exists locally** → `git checkout <branch>`.
   c. **Branch absent** → `git fetch origin main && git checkout -b <branch> origin/main`.
   d. Verify `git branch --show-current` equals `<branch>`. Mismatch → `AskUserQuestion` and stop.

### Step 3 — Verify symlink

1. Run `test -L plans/<branch>`.
2. Absent or not symlink → create:
   ```
   mkdir -p plans && ln -sf ~/.claude/tasks/<branch> plans/<branch>
   ```
3. Verify: `readlink plans/<branch>` must return path ending in `.claude/tasks/<branch>`. Fails → `AskUserQuestion` and stop.

> **Note:** Task list scoping handled automatically by `TeamCreate` in Phase 2, Step 2. `TeamCreate` with `team_name: "<branch>"` creates task list at `~/.claude/tasks/<branch>/` and sets `CLAUDE_CODE_TEAM_NAME` on all teammates. No manual `export` of `CLAUDE_CODE_TASK_LIST_ID` needed.

### Step 4 — Detect fresh start vs. resume

Check `plans/<branch>-team-state.json`:
- **Absent** → Read `references/phase-1-fresh-start.md` and follow in full. Return here, proceed to Phase 2.
- **Present** → Read `references/phase-1-resume.md` and follow in full. Return here and proceed to Phase 2 (or stop if user declines).

______________________________________________________________________

## Phase 2: Team Creation and Coordination

### Step 1 — Role analysis

1. Read `plans/<branch>-team-state.json`.
2. Collect distinct `role_hint` values across all tasks with `status: pending`. Call this `roles_needed`.
3. Always add `hyperteam-reviewer` and `hyperteam-worker` to `roles_needed` (reviewer always needed; worker is fallback for unmatched hints).

### Step 2 — Create the team

Call `TeamCreate` with:
- Team name: `<branch>`
- One teammate per role in `roles_needed`
- Prompt includes: branch name, paths to `plans/<branch>-team-state.json`, `plans/<branch>-progress.txt`, `plans/<branch>-prd.md`

### Step 3 — Seed the native task list

For every task in `team-state.json` with `status: pending`:

1. Call `TaskCreate` with YAML front-matter block + full story text as `description`:
   ```
   ---
   id: <task_id>
   type: <FEAT|DOC|GATE>
   role_hint: <role_hint>
   blocked_by:
     - <blocker_id_1>
     - <blocker_id_2>
   ---

   <full story text and acceptance criteria from team-state.json task description>
   ```
2. Store returned task UUID as `native_task_id` in corresponding task in `team-state.json`.
3. After all pending tasks processed, write updated `team-state.json` to disk.

### Step 4 — Broadcast kickoff

Send broadcast `SendMessage` to team:

> Hyperteam `<branch>` is starting.
> State file: `plans/<branch>-team-state.json`
> Progress log: `plans/<branch>-progress.txt`
>
> All specialists: claim tasks from native task list. Parse YAML front-matter in each task's description for `role_hint` and `blocked_by`. Resolve blockers via `team-state.json` (blocker terminal when status `validated` or `completed`).
>
> Reviewer: begin scanning `team-state.json` for completed FEAT tasks with `reviewed: false` immediately.

### Step 5 — Monitor

Main thread monitors run. Lead agent (dispatched in Step 2) handles coordination: review outcomes, failure resets, blocker broadcasts, GATE readiness detection.

React to events:
- **`SendMessage` from lead signalling GATE PASS** → proceed to Phase 4.
- **`SendMessage` from any teammate requiring main-thread intervention** → address and resume.

Main thread does **not** dispatch individual workers or validators. Teammates self-claim.

______________________________________________________________________

## Phase 3: Back-Pressure Gate

> Runs **inside** reviewer agent — not main thread. Reviewer claims GATE native task when lead broadcasts GATE OPEN. See `references/gate-task-template.md` for full gate agent instructions.

Lead notifies main thread only after GATE passes. Proceed to Phase 4.

______________________________________________________________________

## Phase 4: Completion and PR Offer

Read `references/phase-4-completion.md` and follow in full.

______________________________________________________________________

## Phase 5: Team Cleanup

After Phase 4 completes (summary written, PR offered/created/declined), call `TeamDelete` for team `<branch>`. Removes all shared team resources. Must be done after Phase 4 so all teammates fully idle before cleanup.
