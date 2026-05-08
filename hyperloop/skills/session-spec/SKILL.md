---
name: session-spec
description: "Single-pass session-spec generator. Outputs plans/<branch>-session-spec.md ready for /hyperteam."
argument-hint: "<feature description or external sources (e.g., plans/auth-seedling.md, GitHub/JIRA issue, URL(s), etc.)>"
user-invocable: true
disable-model-invocation: true
---

# Session-Spec Generator

Produces `plans/<branch>-session-spec.md` in step→verify format. Single-pass: one interview round, one conflict check, one output.

______________________________________________________________________

## Adversarial Review Mandate

**Primary job: critique requirements, not agree with them.** Before accepting any requirement, actively search for:

1. **Explicit conflicts** — contradictory requirements, mutually exclusive acceptance criteria.
2. **Implicit conflicts vs. codebase** — contradictions with existing ADRs, `CLAUDE.md` patterns, domain model, `CONTRIBUTING.md`. Read `CLAUDE.md` for ADR locations, scan ADR dirs, search project source dirs before accepting any requirement.
3. **Ambiguity-hidden conflicts** — vague requirements that seem compatible but force contradictory impl choices.

Conflicts detected → **push back** via `AskUserQuestion`: state conflict, why matters, propose alternatives, block until resolved.

> **Seedling philosophy:** Seedling doc gives head start — not sacred. Challenge it same as any input.

______________________________________________________________________

## The Job

### Step 1 — Environment Setup

1. Read `$ARGUMENTS`. Empty → `AskUserQuestion` to gather input before proceeding.
2. **Detect input type:**
   - `$ARGUMENTS` is path to existing `.md` file → **seedling mode** (use as baseline draft).
   - Otherwise → **text description mode** (generate from scratch).
3. Derive `<slug>` as short lowercase-kebab-case label from title or description.
4. Generate `<branch>` as `feat-<slug>`.
5. **Detect GitHub issue references:**
   - Scan `$ARGUMENTS` for URLs matching `https://github.com/{owner}/{repo}/issues/{N}`.
   - Matches → `<source_issues>`: list of `owner/repo#N`. No match → `<source_issues>` null.
   - Per issue in `<source_issues>`:
     ```
     gh issue edit <N> --repo <owner>/<repo> --add-assignee @me
     ```
     Fails → print `⚠ Warning: could not assign issue — <error>`. Do **NOT** block spec creation.
6. **Sync main from origin:**
   1. Run `git fetch origin main`.
   2. `git log main..origin/main --oneline` — commits listed → main behind. `AskUserQuestion` to surface; **stop** until user confirms.
7. **Create and checkout branch:**
   ```
   git checkout -B <branch> main
   ```
   Verify `git branch --show-current` equals `<branch>`. Mismatch → `AskUserQuestion` and stop.
8. Create `plans/` if absent: `mkdir -p plans`
9. Create symlink:
   ```
   ln -sf ~/.claude/tasks/<branch> plans/<branch>
   ```
   Verify `test -L plans/<branch>` and `readlink` ends in `.claude/tasks/<branch>`. Fails → `AskUserQuestion` and stop.

### Step 2 — Conflict Scan

Read `CLAUDE.md` for ADR locations. Scan ADR dirs and relevant source dirs for existing code related to requested feature. Identify contradictions between requested and existing. **Mandatory in both modes.**

### Step 3 — Single Focused Interview

Max 2–3 `AskUserQuestion` calls total. Rules:

- At least one question must address conflicts found in Step 2 (if any).
- No conflicts found → ask about goals, scope boundaries, success criteria.
- Do not re-ask content user already provided.

**Seedling mode:** ask 2–3 targeted questions on gaps, conflicts, ambiguities — not repeating seedling content.

**Text description mode:** ask 2–3 questions on problem/goal, scope/boundaries, success criteria.

### Step 4 — Generate Spec

1. If `plans/<branch>-session-spec.md` exists → move to `plans/archive/<branch>-session-spec.md` before writing.
2. Write `plans/<branch>-session-spec.md` in step→verify format per `references/example-session-spec.md`.
3. Spec structure:
   - `<source_issues>` non-null → write metadata table **immediately after H1 and before `## Goal`**, one `| Source Issue |` row per issue.
   - `<source_issues>` null → omit table. H1 followed directly by `## Goal`.
   - Steps framing: each deliverable = `### STEP-<slug>-NN: <name>` with `**Acceptance Criteria:**` checklist.
   - **One step = one commit.** Scope each step so it can be implemented and committed independently (assuming prior steps already on branch). Steps that cannot be committed in isolation must be merged or re-scoped.
   - AC per step: `- [ ]` items — concrete, independently falsifiable checks. Include: artifact exists, behavior correct, project verification command passes. For new API surface, first step creates stubs with failing tests; subsequent steps implement against stable contracts.

4. **Skill assignment per step.** Every step must declare `role_hint` and `skills`. These appear in the task YAML front-matter block when hyperteam seeds the native task list.

   All steps: `role_hint: hyperteam-worker`. No specialist role_hints.

   Assign `skills:` using this mapping (can assign multiple):

   | Task type / signals                                | skills                        |
   | -------------------------------------------------- | ----------------------------- |
   | Python implementation, tests, refactor             | `tdd`, `python`               |
   | TypeScript implementation, tests, refactor         | `tdd`, `typescript`           |
   | API scaffold, endpoint stub, schema generation     | `api-scaffold`                |
   | Docs, README, changelog, ADR, user-facing writing  | `tech-writing`                |
   | Mixed Python + API scaffold                        | `tdd`, `python`, `api-scaffold` |
   | Unknown / untyped / general                        | `tdd`                         |

   Default fallback: `tdd` for any step not matching the table above.

   Each step annotation (embedded in spec body, parsed by hyperteam Phase 1):
   ```
   > skills: tdd, python
   > role_hint: hyperteam-worker
   ```

   Example step with skill annotation:
   ```markdown
   ### STEP-auth-01: Implement JWT validation middleware

   > skills: tdd, python
   > role_hint: hyperteam-worker

   **Acceptance Criteria:**
   - [ ] `src/auth/middleware.py` exists with `validate_jwt(token: str) -> Claims` function
   - [ ] Unit tests in `tests/test_middleware.py` cover valid, expired, malformed token cases
   - [ ] `pre-commit run --all-files` exits 0
   ```

   The hyperteam skill (Phase 2, Step 3) reads these annotations and produces the native task YAML:
   ```yaml
   ---
   id: FEAT-auth-01
   type: FEAT
   role_hint: hyperteam-worker
   skills:
     - tdd
     - python
   blocked_by: []
   ---
   ```

   > **Reading note for agents:** Metadata table (if present) appears **immediately after H1** and **before first `##` section**. Parsers: locate H1, scan forward collecting all `| Source Issue |` rows before `##`; none found → `source_issues` is `null`.

### Step 5 — Final Conflict Sweep

Verify: no step contradicts another; no step conflicts with codebase findings from Step 2. Conflict found → raise with user via `AskUserQuestion` and resolve before saving.

> **Do NOT start implementing. Create spec only.**

______________________________________________________________________

## Before Saving

- [ ] Step 1 complete: `<branch>` chosen, main synced, branch checked out and verified, `plans/` exists, symlink created and validated
- [ ] GitHub issue URLs scanned: `<source_issues>` set to `["owner/repo#N", ...]` or `null`; `gh issue edit` assignment attempted per issue (warning on failure, spec not blocked)
- [ ] Input mode detected: seedling or text description
- [ ] `CLAUDE.md` ADR locations, ADR dirs, project source dirs searched for conflicts (Step 2)
- [ ] Interview complete: ≤3 `AskUserQuestion` calls; ≥1 addressed Step 2 conflicts (if any)
- [ ] `<source_issues>` non-null → metadata table present immediately after H1 and before `## Goal`; null → no table
- [ ] Spec uses `## Goal / ## Context / ## Non-goals / ## Steps / ## Open Questions` structure
- [ ] Each step has `**Acceptance Criteria:**` checklist with ≥1 concrete, independently falsifiable `- [ ]` item
- [ ] Old `plans/<branch>-session-spec.md` archived to `plans/archive/` if existed
- [ ] Final conflict sweep complete — no intra-spec contradictions, no codebase conflicts
