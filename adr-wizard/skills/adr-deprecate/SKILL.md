---
name: adr-deprecate
description: "This skill should be used when the user asks to 'deprecate an ADR', 'mark an ADR as obsolete', 'retire an architectural decision', 'flag an ADR as no longer applicable', or when an existing architectural decision is no longer relevant and should be marked as deprecated while preserving history."
argument-hint: "ADR number to deprecate and why (e.g., '5 No longer applicable after migration to microservices')"
user-invocable: true
---

# adr-deprecate

Marks existing ADR as deprecated, records reason in file, updates dir README.md index.

> ADRs are **additive only**: never delete or heavily rewrite accepted ADR. Skill preserves history — old ADR remains readable, new ADR explains what changed and why.

---

## Step 1 — Discover ADR directories

Follow same discovery as `adr-create` (Step 1):

1. Search `CLAUDE.md` for heading containing `ADR Locations`. Collect bullet paths into `adr_dirs`, stripping inline `# comments`.
2. Not found → fall back to scanning for `docs/adrs`, `decisions`, `architecture/decisions`.
3. Empty → inform user and stop.

## Step 2 — Select target directory

Multiple dirs → ask user which contains ADR to deprecate, or auto-suggest most relevant based on recent file context (same logic as `adr-create` Step 2). Wait for confirmation.

## Step 3 — Identify the ADR to deprecate

1. List all files in `target_dir` matching `NNNN-*.md`.
2. Skill invoked with ADR number (e.g., `/adr-deprecate 3`) → use directly: find file matching `0003-*.md`.
3. Otherwise, display list of current ADRs with numbers, titles, statuses, and ask:
   > Which ADR do you want to deprecate? (enter ADR number or filename)
4. Confirm selection:
   > Deprecating: ADR-NNNN — <title>. Proceed? (y/n)
5. ADR status already `Deprecated` → warn and stop:
   > ADR-NNNN is already deprecated.

## Step 4 — Understand and draft the deprecation

Skill must understand **why** ADR is deprecated and produce clear, informative deprecation note. Gather from multiple sources:

1. **Read ADR in full** — understand what decision it records and current consequences.
2. **Check conversation context** — user may have discussed why decision is obsolete, what replaced it, or what changed.
3. **Check arg** — if user provided text after ADR number (e.g., `/adr-deprecate 3 migrated to microservices`), use as deprecation rationale.

Draft:
- `deprecation_reason`: concise one-line summary for `**Deprecated:**` metadata field (e.g., "No longer applicable after migration to microservices architecture").
- `deprecation_note`: 2–4 sentence paragraph to append to ADR's `## Context` (or new `## Deprecation Note` section) explaining:
  - What changed making decision no longer relevant
  - Whether replacement decision exists (reference by ADR number if so)
  - Any cleanup or migration resulting from deprecation

Reason can't be confidently inferred → interview user with `AskUserQuestion`:

> I'm deprecating ADR-NNNN (<title>). To write proper deprecation record, I need:
>
> 1. Why is this decision no longer relevant? (What changed?)
> 2. Has it been replaced by another decision, or simply obsolete?
> 3. Was any cleanup or migration done as result?

Batch all needed questions into single `AskUserQuestion` call. Proceed only after reason and note are clear.

## Step 5 — Update the ADR file

1. Open ADR file.
2. Find `**Status:**` line. Replace value with `Deprecated`.
3. After `**Status:**` line (or after `**Date:**` if Status comes before Date), add:
   ```
   **Deprecated:** <deprecation_reason>
   ```
4. Append `## Deprecation Note` section at end of file (before trailing blank lines) with `deprecation_note`.
5. Save file.

## Step 6 — Update the index

1. Open `<target_dir>/README.md`.
2. Find row for this ADR in table (match by filename or ADR number).
3. Update `Status` column to `Deprecated`.
4. Save file.

## Step 7 — Confirm

Inform user:

> Deprecated ADR-NNNN: <title>
> Reason: <deprecation_reason>
> Updated: `<target_dir>/<NNNN>-<slug>.md`
> Updated index: `<target_dir>/README.md`

## Step 8 — Post-write validation

Invoke `adr-check` in scoped mode against deprecated ADR file:

```
/adr-check <deprecated_adr_path>
```

Display all output to user.

- **Structural FAIL:** Block completion, prompt user to resolve issue before proceeding:
  > The deprecated ADR has a structural validation failure. Please fix the issue above before confirming this deprecation is complete.
- **Style warnings only:** Display and continue. Style warnings are informational, don't block completion.
