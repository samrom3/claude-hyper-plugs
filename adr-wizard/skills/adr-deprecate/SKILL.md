---
name: adr-deprecate
description: "Deprecate an existing Architecture Decision Record (ADR) that is no longer relevant, recording the reason while preserving history. Use when an architectural decision, design choice, or technical decision is obsolete or no longer applicable. Invocable via /adr-deprecate."
user-invocable: true
---

# adr-deprecate

Marks an existing ADR as deprecated, records the reason in the file, and updates the directory's
README.md index.

---

## Step 1 — Discover ADR directories

Follow the same discovery procedure as `adr-create` (Step 1):

1. Search `CLAUDE.md` for any heading containing `ADR Locations`. Collect bullet paths into
   `adr_dirs`, stripping inline `# comments`.
2. If not found, fall back to scanning for `docs/adrs`, `decisions`, `architecture/decisions`.
3. If empty, inform the user and stop.

## Step 2 — Select target directory

If multiple directories exist, ask the user which directory contains the ADR to deprecate, or
auto-suggest the most relevant directory based on recent file context (same logic as `adr-create`
Step 2). Wait for confirmation.

## Step 3 — Identify the ADR to deprecate

1. List all files in `target_dir` matching `NNNN-*.md`.
2. If the user invoked the skill with an ADR number (e.g., `/adr-deprecate 003`), use it
   directly: find the file matching `0003-*.md`.
3. Otherwise, display the list of current ADRs with their numbers, titles, and statuses, and
   ask:
   > Which ADR do you want to deprecate? (enter the ADR number or filename)
4. Confirm the selection:
   > Deprecating: ADR-NNNN — <title>. Proceed? (y/n)
5. If the ADR's current status is already `Deprecated`, warn the user and stop:
   > ADR-NNNN is already deprecated.

## Step 4 — Get deprecation reason

Ask the user:

> Why is this ADR being deprecated? (e.g., "No longer applicable after migration to
> microservices")

Use this as `deprecation_reason`.

## Step 5 — Update the ADR file

1. Open the ADR file.
2. Find the `**Status:**` line. Replace its value with `Deprecated`.
3. After the `**Status:**` line (or after `**Date:**` if Status comes before Date), add:
   ```
   **Deprecated:** <deprecation_reason>
   ```
4. Save the file.

## Step 6 — Update the index

1. Open `<target_dir>/README.md`.
2. Find the row for this ADR in the table (match by filename or ADR number).
3. Update the `Status` column value in that row to `Deprecated`.
4. Save the file.

## Step 7 — Confirm

Inform the user:

> Deprecated ADR-NNNN: <title>
> Reason: <deprecation_reason>
> Updated: `<target_dir>/<NNNN>-<slug>.md`
> Updated index: `<target_dir>/README.md`
