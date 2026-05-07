---
name: adr-create
description: "This skill should be used when the user asks to 'create an ADR', 'document an architectural decision', 'record a design decision', 'write an architecture decision record', or when the model identifies an architectural decision in context that warrants formal documentation. Evaluates decision worthiness before authoring and guides section drafting."
argument-hint: "Brief description of the architectural decision (e.g., 'Use PostgreSQL for primary storage')"
user-invocable: true
---

# adr-create

Creates new ADR file with auto-numbering, fills Nygard template, updates dir's README.md index.

---

## Step 0 — Evaluate decision worthiness

### Phase A — Silent evaluation (always runs first)

Assess decision across axes before engaging user:

| Axis | What to assess |
|------|----------------|
| **Alternatives** | Obvious alternatives exist? Or was decision effectively forced? |
| **Scope** | Crosses module/team/service boundaries — or local to one fn/file? |
| **Reversibility** | Undoing costs significant? (data migration, API contract change, retraining) |
| **Tradeoffs** | Context surface what was given up? |
| **Existing ADRs** | Extends, contradicts, or supersedes prior choice? |

**Clear verdict → act directly:**
- **Clear yes** (cross-cutting, meaningful alternatives, non-trivial reversal cost, or intersects existing ADR): proceed to Step 1 without engaging user.
- **Clear no** (routine impl detail, derivable from code, no alternatives in play, entirely local): inform user, suggest lighter alternative (inline comment, team note), stop. No escape hatch.

**Inconclusive** (alternatives ambiguous, tradeoffs not surfaced, decision boundaries unclear) → Phase B.

---

### Phase B — Adversarial debate (only when Phase A inconclusive)

Engage user to extract info Phase A couldn't resolve. Posture: adversarial-constructive. Argue against ADR necessity or framing until clear. State challenges directly; no softening.

**Challenge axes** (raise 1–3 based on what Phase A left unresolved):

| Axis | Counterpoint |
|------|-------------|
| **Scope** | "This sounds contained to [X]. Why does someone on a different part of the system six months from now need to know?" |
| **Obviousness** | "Experienced engineer reading the code would likely infer this from [Y]. What would they miss?" |
| **Reversibility** | "Real cost of undoing this? If low, permanent record may not be warranted." |
| **Alternatives** | "What alternatives did you actually consider? If only one option was ever on the table, this may be recording an outcome, not a decision." |
| **Novelty** | "Is this consistent with existing ADR? If so, it may be implementing an already-approved pattern." |

1–2 exchanges. Steelman responses, counter if still unconvincing. End when picture is clear.

---

### Outcome routing (after Phase A or B)

| Outcome | Action |
|---------|--------|
| **Single ADR warranted** | Proceed to Step 1. |
| **Multiple ADRs warranted** | Surface each distinct decision: "This looks like two separate decisions: [A] and [B]. I'll create them in sequence — starting with [A]." Proceed to Step 1 for first; loop Step 0 for each subsequent. |
| **Supersede/deprecate warranted** | "This appears to supersede ADR-NNNN ([title]). I'll hand off to `/adr-supersede`." Invoke lifecycle skill, stop. |
| **No ADR warranted** | Explain confidently. Phase A clear no → stop, no escape hatch. Debate concludes no → offer escape hatch via `AskUserQuestion`: **Proceed anyway** / **Revisit framing**. Revisit restarts Phase B; proceed → Step 1. |

## Step 1 — Discover ADR directories

1. Read project `CLAUDE.md`.
2. Search for heading containing `ADR Locations` (case-insensitive, any heading level).
3. If found, collect bullet items as relative paths, strip inline `# comments`. Store as `adr_dirs`.
4. If not found, fall back: scan repo root for `docs/adrs`, `decisions`, `architecture/decisions`.
5. If `adr_dirs` empty, offer bootstrap via `AskUserQuestion`:
   > No ADR directories found. Would you like me to create one?

   Options: `docs/adrs/` (recommended) or custom path.

   On confirm:
   a. Create dir.
   b. Copy [`references/README-template.md`](references/README-template.md) → `<dir>/README.md`, replace `<PROJECT_NAME>`.
   c. Copy [`references/0000-adr-template.md`](references/0000-adr-template.md) → new dir.
   d. Add `### ADR Locations` section to `CLAUDE.md` (or create if absent).
   e. Set `adr_dirs` to new dir, continue to Step 2.

## Step 2 — Select target directory

1. Single entry → use as `target_dir`.
2. Multiple entries:
   a. Identify dir whose path best matches recent file context (longest common prefix with files mentioned in conversation/editor).
   b. Present suggestion:
      > Suggested ADR directory: `<target_dir>` (based on recent file context)
      > Other options: `<list remaining dirs>`
      > Press Enter to accept, or type path of another directory.
   c. Wait for confirmation. Use confirmed path as `target_dir`.

## Step 3 — Determine next ADR number

1. List files in `target_dir` matching `NNNN-*.md`.
2. Find highest number.
3. `next_num = highest + 1` (or `1` if none exist).
4. Zero-pad to 4 digits: `1` → `0001`, `12` → `0012`.

## Step 4 — Parse the decision context

1. **Arg provided** (e.g., `/adr-create Use PostgreSQL for primary storage`): use as `decision_summary`.
2. **No arg, conversation contains clear decision**: extract as `decision_summary`.
3. **No arg, no clear context**: ask:
   > What architectural decision are you recording? (e.g., "Use PostgreSQL for primary storage")

From `decision_summary` + conversation context, derive `adr_title`. Convert to kebab-case slug: lowercase, spaces/specials → hyphens.

## Step 5 — Draft all ADR sections

Author every section — no template placeholders for user. Infer from conversation, codebase, `decision_summary`. Use `AskUserQuestion` when info insufficient.

### Context

*Context: describe problem forces, not solution. Reader understands why decision needed, not pre-sold on answer.*

Draw from conversation history, recent code/files, constraints/tradeoffs mentioned. If insufficient:
> What problem or situation prompted this decision? What constraints are in play?

### Decision

*Decision: active voice ("We decided to X"). Specific, declarative. No passive constructions. State what was chosen, not why or what happens next.*

Be specific (e.g., "We will use PostgreSQL 16 as primary data store for all user-facing services"). Draw from `decision_summary` + explicit conversation choices.

If ambiguous:
> Can you state the decision more precisely? What exactly are we choosing to do?

### Consequences

*Consequences: ≥1 genuinely adverse consequence required. 200–400 words. Substantially more → consider splitting into two ADRs.*

3–7 bullets: expected benefits, trade-offs/risks (≥1 required), follow-up work, migration/compatibility implications.

If can't infer:
> What are the main benefits and trade-offs of this decision? Any follow-up work it creates?

### Interview flow

Batch questions — ask multiple sections' questions in single `AskUserQuestion`. Proceed to file creation only after all sections have substantive content.

## Step 6 — Create the ADR file

1. Filename: `<target_dir>/<NNNN>-<slug>.md`.
2. Fill template from [`references/0000-adr-template.md`](references/0000-adr-template.md):
   - Replace `NNNN` with zero-padded number.
   - Replace `Title` with `adr_title`.
   - Set `**Status:** Proposed` (new ADRs always Proposed unless user specifies otherwise).
   - Set `**Date:**` to today's date (`YYYY-MM-DD`).
   - Write Context, Decision, Consequences content.
3. Write file to disk.

## Step 7 — Update the index

1. If `<target_dir>/README.md` missing, create:
   ```markdown
   # Architecture Decision Records

   | ADR | Title | Status |
   |-----|-------|--------|
   ```
2. Add row:
   ```
   | [ADR-NNNN](<NNNN>-<slug>.md) | <adr_title> | Proposed |
   ```
   Insert after last existing row.
3. Save `README.md`.

## Step 8 — Confirm

> Created `<target_dir>/<NNNN>-<slug>.md` (ADR-NNNN: <adr_title>)
> Updated index: `<target_dir>/README.md`
>
> Review the ADR and change the Status to `Accepted` when the decision is finalised.
>
> Tip: commit this ADR in the same PR as (or just before) the code it describes so the decision and implementation are traceable together.

## Step 9 — Post-write validation

1. Invoke `adr-check` scoped against new ADR: `adr-check <target_dir>/<NNNN>-<slug>.md`
2. Review result:
   - **Structural FAIL:** Block completion, present issues to user. Offer to fix directly if straightforward.
   - **Style warnings:** Display, don't block. ADR complete; user may address or not.
   - **PASS (no warnings):** Complete successfully.
