---
name: adr-supersede
description: "This skill should be used when the user asks to 'supersede an ADR', 'replace an architectural decision', 'update an ADR with a new decision', 'mark an ADR as superseded', or when an existing architectural decision has changed and needs to be replaced while preserving the historical record."
argument-hint: "Old ADR number, optional arrow to existing new ADR number, and the new decision or reason (e.g., '3 Switching from PostgreSQL to CockroachDB for horizontal scaling' or '3->7 CockroachDB chosen to replace PostgreSQL')"
user-invocable: true
---

# adr-supersede

Creates new ADR superseding existing one, updates old ADR's status, maintains bidirectional cross-refs in both files and dir index.

> ADRs **additive only**: never delete or heavily rewrite accepted ADR. History preserved — old ADR stays readable.

---

## Step 1 — Discover ADR directories

Same discovery as `adr-create` Step 1:
1. Search `CLAUDE.md` for heading containing `ADR Locations`. Collect bullet paths into `adr_dirs`, strip inline `# comments`.
2. If not found, fall back: scan for `docs/adrs`, `decisions`, `architecture/decisions`.
3. If empty, inform user and stop.

## Step 2 — Select target directory

Multiple dirs → auto-suggest based on recent file context (same as `adr-create` Step 2). Present suggestion, wait for confirmation. Use confirmed path as `target_dir`.

## Step 3 — Parse the argument and identify ADRs

Format:
```
<old_num>[->new_num] [new decision text or reason]
```

Examples:
- `3 Switching from PostgreSQL to CockroachDB` → old=3, new ADR doesn't exist yet, decision text provided
- `3->7 CockroachDB chosen to replace PostgreSQL` → old=3, new=7 (ADR-0007 already exists), rationale provided
- `3->7` → old=3, new=7 already exists, no extra text

Resolution:
1. List files in `target_dir` matching `NNNN-*.md`.
2. **Parse `old_num`**. Zero-pad to 4 digits, find file as `old_adr`. No arg → display list and ask:
   > Which ADR is being superseded? (enter the number)
3. **Parse `new_num`** if `->` present:
   - `new_num` provided + file exists → set `new_adr`, skip Steps 4–6, go to Step 7.
   - `new_num` provided + file missing → treat as title/number hint for Step 6.
   - No `->` → create new ADR (proceed through all steps).
4. Confirm: `Superseding: ADR-NNNN — <title>. Proceed? (y/n)`
5. `old_adr` already `Superseded by ADR-MMMM` → warn: `ADR-NNNN is already superseded by ADR-MMMM. Supersede again? (y/n)`. Proceed only on confirm.

## Step 4 — Understand the supersession

Gather from:
1. **Read old ADR in full** — Context, Decision, Consequences as baseline.
2. **Conversation context** — why old approach no longer works, what replacement is.
3. **Arg** — text after `/adr-supersede` used as supersession rationale.

Derive:
- `new_adr_title`: concise title for replacement.
- `what_changed`: why original replaced (new constraints, better alternatives, lessons learned).
- `new_decision`: new decision, specific and declarative.

If any can't be inferred, ask via `AskUserQuestion`. Batch questions:
> I'm superseding ADR-NNNN (<old_title>). To write replacement ADR:
>
> 1. What is the new decision? (e.g., "Use Redis for session storage instead of PostgreSQL")
> 2. What changed that makes the old decision no longer appropriate?

Proceed only after all three pieces clear.

## Step 5 — Draft the new ADR sections

Use old ADR as foundation + supersession context from Step 4.

### Context

Start from old ADR's Context (original problem), then explain what changed (new constraints, growth, incidents, technology shifts, lessons learned that invalidate original decision). Reference old ADR by number.

### Decision

New decision, clear and declarative. Contrast with old where helpful (e.g., "We will migrate from X to Y"). Include `**Supersedes:** ADR-<old_num>` cross-ref.

### Consequences

3–7 bullets: benefits of new over old, migration/transition costs, risks/tradeoffs of new decision, follow-up work, what becomes easier/harder vs. superseded approach.

If can't infer:
> What are key benefits of new approach, and what migration/transition costs are anticipated?

## Step 6 — Create the new ADR via adr-create

Invoke `adr-create` with:
- `new_adr_title` as decision summary arg
- Full supersession context (old ADR content + what changed + new decision) for section drafting

After completion, capture new file path and number as `new_adr` and `next_num`.

Open new ADR file, add cross-ref after `**Date:**`:
```
**Supersedes:** ADR-<old_num_padded>
```
Save file.

## Step 7 — Link the superseded ADR

1. Open `old_adr`.
2. Find `**Status:**`. Replace value with `Superseded by ADR-<next_num_padded>`.
3. If `->` path (new_adr already existed): add `**Superseded by:**` metadata after `**Date:**` if not present.
4. Save file.

## Step 8 — Update the index

1. Open `<target_dir>/README.md`.
2. Find row for old ADR. Update `Status` to `Superseded by ADR-<next_num_padded>`.
3. New ADR just created by adr-create → index row already added, skip. New ADR already existed and not in index → add now.
4. Save `README.md`.

## Step 9 — Confirm

> ADR-<old_num_padded> → ADR-<next_num_padded>: <new_adr_title>
> Updated ADR-<old_num_padded>: Status → "Superseded by ADR-<next_num_padded>"
> Updated index: `<target_dir>/README.md`
>
> Review the new ADR and change the Status to `Accepted` when finalised.

## Step 10 — Post-write validation

Invoke `adr-check` scoped against superseded ADR (file modified in Step 7):

```
/adr-check <old_adr_path>
```

Display all output to user.

- **Structural FAIL:** Block completion, prompt user to resolve:
  > The superseded ADR has a structural validation failure. Please fix the issue above before confirming this supersession is complete.
- **Style warnings only:** Display and continue. Not blocking.

Note: new ADR (created via `adr-create` in Step 6) already validated by adr-create's post-write step — don't run `adr-check` again.
