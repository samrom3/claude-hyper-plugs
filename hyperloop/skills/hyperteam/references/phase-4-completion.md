# Phase 4 — Completion and PR Offer

Runs after lead returns and GATE task has passed.

---

## Step 1 — Compute completion summary

Read `plans/<branch>-progress.txt` and `plans/<branch>-team-state.json`. Report:
- Total tasks completed, by type: FEAT, DOC, GATE.
- FEAT tasks validated by reviewer.
- Gate iterations (from `gate_iterations` in `team-state.json`).
- Time elapsed: first `progress.txt` entry to last.

---

## Step 2 — Mark run complete

Update `plans/<branch>-team-state.json`: set `metadata.status` to `"complete"`.

Append to `plans/<branch>-progress.txt`:

```
## [ISO timestamp] - COMPLETE
- All tasks validated and gate passed.
- Total tasks: N (FEAT: X, DOC: Y, GATE: Z)
- Gate iterations: N
- Time elapsed: [computed duration]
---
```

---

## Step 3 — Offer PR creation

Use `AskUserQuestion`:

> GATE passed. All tasks complete. Create a PR for branch `<branch>`?
> (Title derived from spec H1. Answering 'no' leaves branch open for manual PR.)

---

## Step 4a — On confirmation: create the PR

Run `gh pr create`:

- `--title`: First H1 in `plans/<branch>-session-spec.md` after frontmatter, backtick-wrapped skill name stripped (plain prose title).
- `--body`: Summary including:
  1. **Goal** section from spec (verbatim or abbreviated).
  2. Linked steps: list of `FEAT-*` and `DOC-*` task IDs with titles.
  3. **Source issue close links** (if `metadata.source_issues` non-null and non-empty):
     - Run `gh repo view --json nameWithOwner --jq '.nameWithOwner'` → current repo `owner/repo`.
     - Per `source_issues` entry, emit one `Closes` line:
       - Same repo: `Closes #N`
       - Cross-repo: `Closes https://github.com/owner/repo/issues/N`
     - All `Closes` lines **after** linked steps, **before** `---` footer, each on own line, block surrounded by blank lines.
     - `source_issues` null/empty → omit section entirely.
  4. Standard footer:
     ```
     ---
     🤖 Generated with [Claude Code](https://claude.com/claude-code)
     ```
- `--base main` (or repo's default branch).

---

## Step 4b — On decline

Print: "Branch `<branch>` is ready. Run `gh pr create` manually when you are ready."

Exit cleanly.
