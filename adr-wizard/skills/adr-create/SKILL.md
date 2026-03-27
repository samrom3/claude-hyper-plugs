---
name: adr-create
description: "Create a new Architecture Decision Record (ADR) in the correct directory with auto-numbering and index updates. Triggers when the user or model discusses an architectural decision, design choice, technical decision, or needs to document a new architectural pattern, technology selection, or system design choice. Invocable via /adr-create."
user-invocable: true
---

# adr-create

Creates a new ADR file with auto-numbering, fills it from the Nygard template, and updates the
directory's README.md index.

---

## Step 1 — Discover ADR directories

1. Read the project's `CLAUDE.md`.
2. Search for any heading containing the text `ADR Locations` (case-insensitive, any heading
   level, e.g., `## ADR Locations`, `### ADR Locations`).
3. If found, read every bullet list item under that heading as a relative path. Ignore any inline
   `# comment` annotation after the path (strip from `#` to end of line). Collect all paths into
   `adr_dirs`.
4. If **no** such heading exists in CLAUDE.md, fall back: scan the repository root for
   directories named `docs/adrs`, `decisions`, or `architecture/decisions`. Use whichever exist
   as `adr_dirs`.
5. If `adr_dirs` is empty, inform the user:
   > No ADR directories found. Create one (e.g., `docs/adrs/`) and add it to CLAUDE.md under
   > a `### ADR Locations` heading, then run `/adr-create` again.
   Stop.

## Step 2 — Select target directory

1. If `adr_dirs` has exactly one entry, use it as `target_dir`.
2. If `adr_dirs` has multiple entries:
   a. Identify the directory whose path best matches the user's recent file context. "Recent
      file context" means: any files mentioned in the current conversation, files opened in the
      editor, or the directory containing the file most recently mentioned. Choose the ADR
      directory whose path shares the longest common prefix with that context.
   b. Present this as the suggested default:
      > Suggested ADR directory: `<target_dir>` (based on recent file context)
      > Other options: `<list remaining dirs>`
      > Press Enter to accept, or type the path of another directory.
   c. Wait for user confirmation or override.
   d. Use the confirmed path as `target_dir`.

## Step 3 — Determine next ADR number

1. List all files in `target_dir` matching the pattern `NNN-*.md` (where NNN is 1–3+ digits).
2. Find the highest number among all matching filenames.
3. `next_num = highest + 1` (or `1` if no matching files exist).
4. Zero-pad `next_num` to 3 digits: e.g., `1` → `001`, `12` → `012`.

## Step 4 — Get ADR title

If the user invoked `/adr-create` without a title argument and the conversation context does not
clearly name the decision, ask:

> What is the title of this ADR? (e.g., "Use PostgreSQL for primary storage")

Use the supplied title as `adr_title`. Convert it to a kebab-case slug for the filename:
lowercase, spaces and special characters replaced with hyphens.

## Step 5 — Create the ADR file

1. Construct filename: `<target_dir>/<NNN>-<slug>.md` where NNN is the zero-padded number and
   slug is the kebab-case title.
2. Fill in the template from `references/adr-template.md`:
   - Replace `NNN` with the zero-padded number.
   - Replace `Title` with `adr_title`.
   - Set `**Status:** Proposed` (new ADRs always start as Proposed unless the user specifies
     otherwise).
   - Set `**Date:**` to today's date in `YYYY-MM-DD` format.
   - Leave `## Context`, `## Decision`, and `## Consequences` sections with their guidance text
     so the user can fill them in.
3. Write the file to disk.

## Step 6 — Update the index

1. Check if `<target_dir>/README.md` exists.
   - If it does not exist, create it with this structure:
     ```markdown
     # Architecture Decision Records

     | ADR | Title | Status |
     |-----|-------|--------|
     ```
2. Add a new row to the table in `README.md`:
   ```
   | [ADR-NNN](<NNN>-<slug>.md) | <adr_title> | Proposed |
   ```
   Insert it at the end of the table (after the last existing row).
3. Save `README.md`.

## Step 7 — Confirm

Inform the user:

> Created `<target_dir>/<NNN>-<slug>.md` (ADR-NNN: <adr_title>)
> Updated index: `<target_dir>/README.md`
>
> Fill in the Context, Decision, and Consequences sections, then change the Status to
> `Accepted` when the decision is finalised.
