---
name: adr-supersede
description: "Supersede an existing Architecture Decision Record (ADR) with a new one, preserving history with bidirectional cross-references. Use when an architectural decision, design choice, or technical decision has changed and the old ADR needs to be replaced. Invocable via /adr-supersede."
user-invocable: true
---

# adr-supersede

Creates a new ADR that supersedes an existing one, updates the old ADR's status, and maintains
bidirectional cross-references in both files and the directory index.

---

## Step 1 — Discover ADR directories

Follow the same discovery procedure as `adr-create` (Step 1):

1. Search `CLAUDE.md` for any heading containing `ADR Locations`. Collect bullet paths into
   `adr_dirs`, stripping inline `# comments`.
2. If not found, fall back to scanning for `docs/adrs`, `decisions`, `architecture/decisions`.
3. If empty, inform the user and stop.

## Step 2 — Select target directory

If multiple directories exist, auto-suggest the most relevant based on recent file context
(same logic as `adr-create` Step 2). Present the suggestion and wait for confirmation.

Use the confirmed path as `target_dir`.

## Step 3 — Identify the ADR being superseded

1. List all files in `target_dir` matching `NNN-*.md`.
2. If the user invoked the skill with an ADR number (e.g., `/adr-supersede 003`), use it
   directly: find `003-*.md` as `old_adr`.
3. Otherwise, display the list of ADRs (numbers, titles, statuses) and ask:
   > Which ADR is being superseded? (enter the ADR number or filename)
4. Confirm:
   > Superseding: ADR-NNN — <title>. Proceed? (y/n)
5. If the ADR's current status is `Superseded by ADR-MMM`, warn:
   > ADR-NNN is already superseded by ADR-MMM. Supersede again? (y/n)
   Proceed only on confirmation.

## Step 4 — Get new ADR title

Ask:

> What is the title of the new ADR that supersedes ADR-NNN?
> (e.g., "Use Redis for session storage instead of PostgreSQL")

Use this as `new_adr_title`. The new ADR will replace the decision in `old_adr`.

## Step 5 — Create the new ADR

1. Determine `next_num` using the same auto-numbering logic as `adr-create` Step 3.
2. Construct the filename: `<target_dir>/<next_num>-<slug>.md`.
3. Fill in the template from `adr-create/references/adr-template.md`:
   - Replace `NNN` with `next_num`.
   - Replace `Title` with `new_adr_title`.
   - Set `**Status:** Accepted` (superseding ADRs are accepted immediately).
   - Set `**Date:**` to today's date in `YYYY-MM-DD` format.
   - After the `**Date:**` line, add: `**Supersedes:** ADR-<old_num>` (where `old_num` is the
     zero-padded number of `old_adr`).
   - Leave `## Context`, `## Decision`, `## Consequences` sections with guidance text.
4. Write the new file.

## Step 6 — Update the superseded ADR

1. Open `old_adr`.
2. Find the `**Status:**` line. Replace its value with:
   `Superseded by ADR-<next_num>`
3. Save the file.

## Step 7 — Update the index

1. Open `<target_dir>/README.md`.
2. Find the row for the old ADR. Update its `Status` column to `Superseded by ADR-<next_num>`.
3. Add a new row for the new ADR:
   ```
   | [ADR-<next_num>](<next_num>-<slug>.md) | <new_adr_title> | Accepted |
   ```
4. Save `README.md`.

## Step 8 — Confirm

Inform the user:

> Created ADR-<next_num>: <new_adr_title> (`<target_dir>/<next_num>-<slug>.md`)
> Updated ADR-<old_num>: Status → "Superseded by ADR-<next_num>"
> Updated index: `<target_dir>/README.md`
>
> Fill in the Context, Decision, and Consequences sections of the new ADR.
