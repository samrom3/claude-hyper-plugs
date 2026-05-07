---
name: adr-supersede
description: "This skill should be used when the user asks to 'supersede an ADR', 'replace an architectural decision', 'update an ADR with a new decision', 'mark an ADR as superseded', or when an existing architectural decision has changed and needs to be replaced while preserving the historical record."
argument-hint: "Old ADR number, optional arrow to existing new ADR number, and the new decision or reason (e.g., '3 Switching from PostgreSQL to CockroachDB for horizontal scaling' or '3->7 CockroachDB chosen to replace PostgreSQL')"
user-invocable: true
---

# adr-supersede

Creates new ADR superseding existing one, updates old ADR status, maintains bidirectional cross-references in both files and dir index.

> ADRs are **additive only**: never delete or heavily rewrite accepted ADR. Skill preserves history — old ADR remains readable, new ADR explains what changed and why.

---

## Step 1 — Discover ADR directories

Follow same discovery as `adr-create` (Step 1):

1. Search `CLAUDE.md` for heading containing `ADR Locations`. Collect bullet paths into `adr_dirs`, stripping inline `# comments`.
2. Not found → fall back to scanning for `docs/adrs`, `decisions`, `architecture/decisions`.
3. Empty → inform user and stop.

## Step 2 — Select target directory

Multiple dirs → auto-suggest most relevant based on recent file context (same logic as `adr-create` Step 2). Present suggestion and wait for confirmation.

Use confirmed path as `target_dir`.

## Step 3 — Parse the argument and identify ADRs

Parse arg (text after `/adr-supersede`) using format:

```
<old_num>[->new_num] [new decision text or reason]
```

Examples:
- `3 Switching from PostgreSQL to CockroachDB` → old=3, new ADR doesn't exist yet, decision text provided
- `3->7 CockroachDB chosen to replace PostgreSQL` → old=3, new=7 (ADR-0007 already exists), rationale provided
- `3->7` → old=3, new=7 already exists, no extra text

Resolution:

1. List all files in `target_dir` matching `NNNN-*.md`.
2. **Parse `old_num`** from arg. Zero-pad to 4 digits, find matching file as `old_adr`. No arg → display ADR list and ask:
   > Which ADR is being superseded? (enter the number)
3. **Parse `new_num`** if arrow (`->`) present:
   - `new_num` provided and file `<new_num_padded>-*.md` exists → set `new_adr` to that file. New ADR already exists — skip Steps 4–6, go directly to Step 7 to link the two ADRs.
   - `new_num` provided but file missing → treat as title/number hint for new ADR to be created in Step 6.
   - No `->` given → new ADR must be created (proceed through all steps).
4. Confirm:
   > Superseding: ADR-NNNN — <title>. Proceed? (y/n)
5. `old_adr` status already `Superseded by ADR-MMMM` → warn:
   > ADR-NNNN is already superseded by ADR-MMMM. Supersede again? (y/n)
   Proceed only on confirmation.

## Step 4 — Understand the supersession

Skill must understand **why** old decision is being replaced and **what** new decision is. Gather from multiple sources:

1. **Read old ADR in full** — Context, Decision, Consequences provide baseline understanding of what is changing.
2. **Check conversation context** — user may have discussed why old approach no longer works or what replacement should be.
3. **Check arg** — text after `/adr-supersede` (beyond ADR number) → use as supersession rationale.

Derive:
- `new_adr_title`: concise title for replacement decision.
- `what_changed`: why original decision is being replaced (new constraints, better alternatives, lessons learned, etc.).
- `new_decision`: what new decision is, stated specifically and declaratively.

Any can't be confidently inferred → interview user with `AskUserQuestion`. Batch questions — if both new title and rationale needed:

> I'm superseding ADR-NNNN (<old_title>). To write replacement ADR, I need:
>
> 1. What is the new decision? (e.g., "Use Redis for session storage instead of PostgreSQL")
> 2. What changed that makes the old decision no longer appropriate?

Proceed only after all three pieces are clear.

## Step 5 — Draft the new ADR sections

Using old ADR as foundation and supersession context from Step 4, draft all sections:

### Context

Write paragraph explaining situation. Start from old ADR's Context (what was original problem?), then explain what changed — new constraints, growth, incidents, technology shifts, or lessons learned invalidating original decision. Reference old ADR by number.

### Decision

Write new decision clearly and declaratively. Contrast with old decision where helpful (e.g., "We will migrate from X to Y because..."). Include `**Supersedes:** ADR-<old_num>` cross-reference.

### Consequences

Write 3–7 bullets covering:
- Benefits of new approach over old
- Migration or transition costs
- Risks or trade-offs of new decision
- Follow-up work required (e.g., "migrate existing data", "update deployment configs")
- What becomes easier vs. harder compared to superseded approach

Consequences can't be inferred → ask:
> What are the key benefits of the new approach, and what migration or transition costs are anticipated?

## Step 6 — Create the new ADR via adr-create

Invoke `adr-create` skill to create new ADR. Pass:
- `new_adr_title` as decision summary arg
- Full supersession context (old ADR content + what changed + new decision) so adr-create can draft all sections correctly

After adr-create completes, it will have written new file and updated index. Capture new ADR path and number as `new_adr` and `next_num`.

Then open new ADR file and add cross-reference metadata line after `**Date:**`:
```
**Supersedes:** ADR-<old_num_padded>
```

Save file.

## Step 7 — Link the superseded ADR

1. Open `old_adr`.
2. Find `**Status:**` line. Replace value with: `Superseded by ADR-<next_num_padded>`
3. `new_adr` already existed (`->` path) → also add `**Superseded by:**` metadata line after `**Date:**` if not already present.
4. Save file.

## Step 8 — Update the index

1. Open `<target_dir>/README.md`.
2. Find row for old ADR. Update `Status` column to `Superseded by ADR-<next_num_padded>`.
3. New ADR just created by adr-create → its index row already added, skip duplicate. New ADR already existed and not yet in index → add now.
4. Save `README.md`.

## Step 9 — Confirm

Inform user:

> ADR-<old_num_padded> → ADR-<next_num_padded>: <new_adr_title>
> Updated ADR-<old_num_padded>: Status → "Superseded by ADR-<next_num_padded>"
> Updated index: `<target_dir>/README.md`
>
> Review the new ADR and change the Status to `Accepted` when finalised.

## Step 10 — Post-write validation

Invoke `adr-check` in scoped mode against superseded ADR (file this skill directly modified in Step 7):

```
/adr-check <old_adr_path>
```

Display all output to user.

- **Structural FAIL:** Block completion, prompt user to resolve issue before proceeding:
  > The superseded ADR has a structural validation failure. Please fix the issue above before confirming this supersession is complete.
- **Style warnings only:** Display and continue. Style warnings are informational, don't block completion.

Note: new ADR (created via `adr-create` in Step 6) is already validated by adr-create's own post-write step — do not run `adr-check` on it again here.
