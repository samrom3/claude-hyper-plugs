# Phase 1 — Resume

Taken when `plans/<branch>-team-state.json` already exists.

---

## Step 1 — Read authoritative state

- Read `plans/<branch>-team-state.json` (authoritative source of truth).
- Read `plans/<branch>-progress.txt` if exists (context only).

---

## Step 2 — Reconcile state

- Tasks with `status: completed` or `status: validated` → done, leave them.
- Tasks with `status: in_progress` → interrupted, reset to `pending`.
- Clear `native_task_id` to `null` on **every** task (pending and reset) — re-seeded in Step 5.
- `team-state.json` wins over `progress.txt` on disagreement.

---

## Step 3 — Render resume summary

Summary listing:
- Done: IDs and titles of all `completed` / `validated` tasks.
- Remaining: IDs and titles of all `pending` tasks (after reset in Step 2).
- Current `gate_iterations` count.

---

## Step 4 — Confirm with user

Use `AskUserQuestion`:

> Found existing run for `<branch>`.
> Completed: N tasks — <list of completed task IDs and titles>
> Remaining: M tasks — <list of remaining task IDs and titles>
> Gate iterations so far: G
>
> Continue with remaining tasks?

---

## Step 5 — Proceed or stop

- **User confirms:**
  1. Write updated `team-state.json` (`in_progress` tasks reset to `pending`, all `native_task_id` cleared to `null`).
  2. Re-seed native task list: per `status: pending` task, call `TaskCreate` with YAML front-matter and full step text as `description`:

     ```
     ---
     id: <task_id>
     type: <FEAT|DOC|GATE>
     skills:
       - <skill_name>
     blocked_by:
       - <blocker_id_1>
     ---

     <full step text and acceptance criteria>
     ```

  3. Store each returned UUID as `native_task_id` in corresponding task object in `team-state.json`.
  4. Write updated `team-state.json` to disk.
  5. Return to SKILL.md, proceed to Phase 2.

- **User declines** — stop. Leave state files unchanged.
