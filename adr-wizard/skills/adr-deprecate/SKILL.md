---
name: adr-deprecate
description: "This skill should be used when the user asks to 'deprecate an ADR', 'mark an ADR as obsolete', 'retire an architectural decision', 'flag an ADR as no longer applicable', or when an existing architectural decision is no longer relevant and should be marked as deprecated while preserving history."
argument-hint: "ADR number to deprecate and why (e.g., '5 No longer applicable after migration to microservices')"
user-invocable: true
---

# adr-deprecate

Marks existing ADR deprecated, records reason, updates dir's README.md index.

> ADRs **additive only**: never delete or heavily rewrite accepted ADR. History preserved — old ADR stays readable.

---

## Step 1 — Discover ADR directories

Same discovery as `adr-create` Step 1:
1. Search `CLAUDE.md` for heading containing `ADR Locations`. Collect bullet paths into `adr_dirs`, strip inline `# comments`.
2. If not found, fall back: scan for `docs/adrs`, `decisions`, `architecture/decisions`.
3. If empty, inform user and stop.

## Step 2 — Select target directory

Multiple dirs → ask user or auto-suggest based on recent file context (same as `adr-create` Step 2). Wait for confirmation.

## Step 3 — Identify the ADR to deprecate

1. List files in `target_dir` matching `NNNN-*.md`.
2. If invoked with ADR number (e.g., `/adr-deprecate 3`): find file matching `0003-*.md`.
3. Otherwise, display list and ask:
   > Which ADR do you want to deprecate? (enter number or filename)
4. Confirm: `Deprecating: ADR-NNNN — <title>. Proceed? (y/n)`
5. Status already `Deprecated` → warn and stop: `ADR-NNNN is already deprecated.`

## Step 4 — Understand and draft the deprecation

Gather context from:
1. **Read ADR in full** — understand what it records and current consequences.
2. **Conversation context** — user may have discussed why decision is obsolete.
3. **Arg** — text after ADR number used as deprecation rationale.

Draft:
- `deprecation_reason`: one-line summary for `**Deprecated:**` field (e.g., "No longer applicable after migration to microservices architecture").
- `deprecation_note`: 2–4 sentences covering: what changed → decision no longer relevant, replacement ADR if any (reference by number), cleanup/migration done.

If reason not confidently inferable, ask via `AskUserQuestion`:
> I'm deprecating ADR-NNNN (<title>). To write proper deprecation record:
>
> 1. Why is this decision no longer relevant? (What changed?)
> 2. Replaced by another decision, or simply obsolete?
> 3. Any cleanup or migration done?

Batch all questions in single call. Proceed only after reason and note are clear.

## Step 5 — Update the ADR file

1. Open ADR file.
2. Find `**Status:**`. Replace value with `Deprecated`.
3. Add after `**Status:**` (or after `**Date:**` if Status precedes Date):
   ```
   **Deprecated:** <deprecation_reason>
   ```
4. Append `## Deprecation Note` section at file end with `deprecation_note`.
5. Save file.

## Step 6 — Update the index

1. Open `<target_dir>/README.md`.
2. Find row for this ADR.
3. Update `Status` column to `Deprecated`.
4. Save file.

## Step 7 — Confirm

> Deprecated ADR-NNNN: <title>
> Reason: <deprecation_reason>
> Updated: `<target_dir>/<NNNN>-<slug>.md`
> Updated index: `<target_dir>/README.md`

## Step 8 — Post-write validation

Invoke `adr-check` scoped against deprecated ADR:

```
/adr-check <deprecated_adr_path>
```

Display all output to user.

- **Structural FAIL:** Block completion, prompt user to resolve before proceeding:
  > The deprecated ADR has a structural validation failure. Please fix the issue above before confirming this deprecation is complete.
- **Style warnings only:** Display and continue. Not blocking.
