---
name: prd
description: "Multi-phase PRD generator. Outputs plans/<branch>-prd.md ready for /hyperteam."
argument-hint: "<feature description or external sources (e.g., plans/auth-seedling.md, GitHub/JIRA issue, URL(s), etc.)>"
user-invocable: true
disable-model-invocation: true
---

# PRD Generator

Creates detailed, actionable PRDs suitable for implementation by junior devs or AI agents.

______________________________________________________________________

## Critical Review Mandate

**Primary job: critique requirements, not agree with them.** Before accepting any requirement, actively search for:

1. **Explicit conflicts** — contradictory requirements, mutually exclusive acceptance criteria.
2. **Implicit conflicts against existing codebase** — contradictions with existing ADRs, `CLAUDE.md` patterns, current domain model, or `CONTRIBUTING.md` conventions. Read `CLAUDE.md` for ADR locations, scan ADR dirs, search project source dirs before accepting any requirement.
   _(Full ADR scan intentional: PRD work is design work.)_
3. **Ambiguity-hidden conflicts** — vague requirements that seem compatible but force contradictory impl choices.

Conflicts detected → **push back**: use `AskUserQuestion` to state conflict, why it matters, propose concrete alternatives, block until user resolves.

Ambiguities hiding conflicts → interview user with clarifying questions via `AskUserQuestion`.

**Do not proceed to next phase until all ambiguities and conflicts resolved.**

> **Seedling philosophy:** Seedling PRD/external doc gives head start — but is not sacred. Challenge seedlings just as any other input.

______________________________________________________________________

## The Job

### Phase 0: Environment Setup

1. Read `$ARGUMENTS`. Empty → use `AskUserQuestion` to gather input.
2. **Detect input type:**
   - `$ARGUMENTS` is path to existing `.md` file → **seedling mode** (use file as baseline draft).
   - Otherwise → **text description mode** (generate from scratch).
3. Derive `<slug>` as short lowercase-kebab-case label:
   - Seedling: from seedling document title.
   - Text: from feature description (e.g., "Account Rollover" → `account-rollover`).
4. Generate `<branch>` as `feat-<slug>` (e.g., `feat-account-rollover`).
5. **Detect GitHub issue references:**
   - Scan `$ARGUMENTS` for all URLs matching `https://github.com/{owner}/{repo}/issues/{N}`.
   - Matches found → collect as `<source_issues>`: list of `owner/repo#N` references.
   - No match → set `<source_issues>` to `null` (metadata table omitted from PRD).
   - For each issue in `<source_issues>`, run:
     ```
     gh issue edit <N> --repo <owner>/<repo> --add-assignee @me
     ```
     Fails → print `⚠ Warning: could not assign issue — <error>` but do **NOT** block PRD creation.
6. **Sync main from origin:**
   1. Run `git fetch origin main`.
   2. Run `git log main..origin/main --oneline` — commits listed → main is behind. Use `AskUserQuestion` to surface, **stop** until user confirms how to proceed.
   3. Proceed only once `main` up to date with `origin/main`.
7. **Create and checkout branch:**
   ```
   git checkout -B <branch> main
   ```
   Verify with `git branch --show-current` — must equal `<branch>`. Mismatch → `AskUserQuestion` and stop.
8. Create `plans/` dir if absent:
   ```
   mkdir -p plans
   ```
9. Create symlink:
   ```
   ln -sf ~/.claude/tasks/<branch> plans/<branch>
   ```
   Verify: `test -L plans/<branch>` passes, `readlink plans/<branch>` returns path ending in `.claude/tasks/<branch>`. Either check fails → `AskUserQuestion` and stop.

### Phase 1: Draft PRD (baseline)

1. If `plans/<branch>-prd.md` exists, move to `plans/archive/<branch>-prd.md` before continuing.

2. **Before generating anything:** Read `CLAUDE.md` for ADR locations, scan ADR dirs, search project source dirs for existing code related to feature. Identify conflicts between requested and existing. Mandatory in both modes.

3. **Branch on input mode:**

   **Seedling mode:**
   - Read seedling file. Preserve author's structure, intent, existing sections.
   - Review for internal conflicts and conflicts against existing codebase (step 2).
   - Use `AskUserQuestion` to ask 2–3 targeted clarifying questions (focus on conflicts, gaps, ambiguities — not repeating seedling content).
   - Expand seedling into complete PRD, filling all missing template sections.

   **Text description mode:**
   - Use `AskUserQuestion` to ask 3–5 clarifying questions (problem/goal, core functionality, scope/boundaries, success criteria). At least one question must probe conflicts with existing functionality from step 2.
   - Generate complete PRD from scratch.

4. Generated PRD must follow annotated example in `references/example-prd.md` and include **Design Considerations** and **Open Questions**.

5. Developer stories: scaffold-first / implement-second for any new/changed API surface. First story creates typed stubs (interfaces, API contracts) with `NotImplementedError` bodies and skeleton tests; subsequent stories implement business logic against stable contracts via TDD.

6. Save to `plans/<branch>-prd.md`.
   - `<source_issues>` non-null and non-empty → write metadata table **immediately after H1 heading** and **before `## 1.`**, one `| Source Issue |` row per issue:

     ```markdown
     # <Title>

     | Field        | Value                          |
     | ------------ | ------------------------------ |
     | Source Issue | owner/repo#N                   |
     | Source Issue | owner/repo#M                   |

     ## 1. Introduction/Overview
     ```

   - `<source_issues>` null or empty → omit metadata table. H1 followed directly by `## 1. Introduction/Overview`.

   > **Reading note for agents/future readers:** Metadata table (if present) appears **immediately after H1** and **before first `##` section**. Parsers: locate H1, scan forward collecting all `| Source Issue |` rows before `##`; none found → `source_issues` is `null`.

### Phase 2: Design refinement

1. Review PRD's **Design Considerations**. Use `AskUserQuestion` for targeted design questions.
2. **Explicitly cross-check** each proposed design choice against existing ADRs and `CLAUDE.md` patterns. Design choice contradicts existing decision → surface as conflict requiring resolution (possibly via new ADR superseding old one).
3. Append newly discovered questions to **bottom of Open Questions** (keep existing; add "Added in Phase 2" subsection).
4. Update `plans/<branch>-prd.md` with refined design considerations and updated open questions.

### Phase 3: Final refinement

1. Use `AskUserQuestion` to ask user **all remaining Open Questions**.
2. Refine PRD one final time based on answers.
3. **Final conflict sweep:** Verify no requirement contradicts another, and no requirement conflicts with existing codebase from Phase 1 research. Conflict found → raise with user and resolve before saving.
4. Save final version to `plans/<branch>-prd.md`.

> **Important:** Do NOT start implementing. Just create the PRD.

______________________________________________________________________

## Before Saving

- [ ] Phase 0 complete: `<branch>` chosen, main synced from origin, branch checked out and verified, `plans/` exists, symlink `plans/<branch>` → `~/.claude/tasks/<branch>` created and validated
- [ ] GitHub issue URLs scanned: `<source_issues>` set to `["owner/repo#N", ...]` or `null`; `gh issue edit` assignment attempted per issue (warning on failure, PRD not blocked)
- [ ] Input mode detected: seedling (file path) or text description
- [ ] `CLAUDE.md` ADR locations, ADR dirs, project source dirs searched for conflicts before generating
- [ ] Phase 1 PRD includes all 9 sections (see `references/example-prd.md`), incl. Design Considerations and Open Questions
- [ ] `<source_issues>` non-null → metadata table present immediately after H1 and before `## 1.`, one row per issue; null → no metadata table
- [ ] Seedling mode: author's structure and intent preserved; only gaps/ambiguities questioned
- [ ] User input gathered each phase as needed; answers incorporated after each refinement
- [ ] Phase 2 cross-checked all design choices against existing ADRs and `CLAUDE.md` patterns
- [ ] Phase 3 final conflict sweep complete — no intra-PRD contradictions, no codebase conflicts
- [ ] Developer stories small, specific, follow scaffold-first pattern
- [ ] Functional requirements numbered (`FR-###`) and unambiguous
- [ ] Old `plans/<branch>-prd.md` archived to `plans/archive/`
- [ ] Non-goals section clarifies Goal section boundaries
- [ ] Newly discovered questions appended to bottom of **Open Questions** (with phase marker)
