---
name: adr-create
description: "This skill should be used when the user asks to 'create an ADR', 'document an architectural decision', 'record a design decision', 'write an architecture decision record', or when the model identifies an architectural decision in context that warrants formal documentation. Evaluates decision worthiness before authoring and guides section drafting."
argument-hint: "Brief description of the architectural decision (e.g., 'Use PostgreSQL for primary storage')"
user-invocable: true
---

# adr-create

Creates new ADR file with auto-numbering, fills from Nygard template, updates dir README.md index.

---

## Step 0 — Evaluate decision worthiness

### Phase A — Silent evaluation (always runs first)

Using domain knowledge and full conversation context, assess decision across these axes before engaging user:

| Axis | What to assess |
|------|---------------|
| **Alternatives** | Do obvious alternatives exist? Would knowledgeable engineer have weighed options, or was this effectively forced? |
| **Scope** | Does decision cross module, team, or service boundaries — or local to one function/file? |
| **Reversibility** | Is cost of undoing significant (data migration, API contract change, team retraining)? |
| **Tradeoffs** | Does conversation context already surface what was given up? |
| **Existing ADRs** | Does decision extend, contradict, or supersede prior architectural choice? |

**If Phase A yields clear verdict**, act directly — no debate needed:

- **Clear yes** (cross-cutting, meaningful alternatives existed, non-trivial reversal cost, or intersects existing ADR): proceed to Step 1 without engaging user.
- **Clear no** (routine impl detail, derivable from code, no alternatives ever in play, entirely local): inform user plainly, suggest lighter alternative (inline comment, team discussion note), stop. No escape hatch.

**If Phase A inconclusive** — alternatives ambiguous, tradeoffs not surfaced, or decision boundaries unclear — proceed to Phase B.

---

### Phase B — Adversarial debate (only when Phase A is inconclusive)

Engage user to extract info Phase A couldn't resolve. Debate loads context Steps 4–5 need; surfaces decision's true shape, which may not match user's original framing.

**Posture: adversarial-constructive.** Argue against ADR necessity — or against framing — until picture is clear. State challenges directly; don't soften into questions with obvious answers.

**Challenge axes** (raise 1–3 based on what Phase A left unresolved):

| Axis | Counterpoint to raise |
|------|-----------------------|
| **Scope** | "This sounds contained to [X]. Why does someone working on different part of system six months from now need to know about it?" |
| **Obviousness** | "Experienced engineer reading code would likely infer this from [Y]. What would they miss that ADR adds?" |
| **Reversibility** | "What is real cost of undoing this? If low, permanent record may not be warranted." |
| **Alternatives** | "What alternatives did you actually consider? If only one option was ever on table, this may be recording outcome rather than decision." |
| **Novelty** | "Is this consistent with existing ADR? If so, may be implementing already-approved pattern rather than making new decision." |

Conduct 1–2 exchanges. Steelman user's responses, then counter if case still unconvincing. Debate ends when picture clear enough to route to outcome.

---

### Outcome routing (after Phase A or Phase B)

| Outcome | Action |
|---------|--------|
| **Single ADR warranted** | Proceed to Step 1. |
| **Multiple ADRs warranted** | Decision is compound. Surface each distinct decision: "This looks like two separate decisions: [A] and [B]. I'll create them in sequence — starting with [A]." Proceed to Step 1 for first; loop back to Step 0 for each subsequent. |
| **Supersede or deprecate warranted** | Decision changes prior architectural choice. Surface: "This appears to supersede ADR-NNNN ([title]). I'll hand off to `/adr-supersede`." Stop and invoke appropriate lifecycle skill. |
| **Decision does not warrant ADR** (Phase A clear no, or debate concludes no) | Explain why confidently. If Phase A source (clear no), stop without escape hatch. If debate concludes no, offer escape hatch: "I'm not convinced this clears the bar for a permanent ADR. That said, you're the author — proceed anyway, or revisit the framing?" Use `AskUserQuestion` with **Proceed anyway** and **Revisit framing**. Revisit restarts Phase B with updated framing; proceed continues to Step 1. |

## Step 1 — Discover ADR directories

1. Read project `CLAUDE.md`.
2. Search for heading containing `ADR Locations` (case-insensitive, any level, e.g., `## ADR Locations`).
3. Found → read every bullet item under that heading as relative path. Ignore inline `# comment` (strip from `#` to EOL). Collect into `adr_dirs`.
4. No such heading → fall back: scan repo root for `docs/adrs`, `decisions`, `architecture/decisions`.
5. `adr_dirs` empty → offer to bootstrap using `AskUserQuestion`:
   > No ADR directories found. Would you like me to create one?

   Options:
   - `docs/adrs/` (recommended — standard location)
   - Other (custom path)

   User confirms:
   a. Create chosen dir.
   b. Copy README template from [`references/README-template.md`](references/README-template.md) into `<chosen_dir>/README.md`, replacing `<PROJECT_NAME>` with actual project name (inferred from repo root dir name or `package.json`/`pyproject.toml`).
   c. Copy [`references/0000-adr-template.md`](references/0000-adr-template.md) into new dir as template reference file.
   d. Add `### ADR Locations` section to project `CLAUDE.md` (or create if absent) with new dir path.
   e. Set `adr_dirs` to newly created dir and continue to Step 2.

## Step 2 — Select target directory

1. `adr_dirs` has exactly one entry → use as `target_dir`.
2. Multiple entries:
   a. Identify dir whose path best matches user's recent file context (files mentioned in conversation, files opened in editor, or dir of most recently mentioned file). Choose ADR dir sharing longest common prefix.
   b. Present as suggested default:
      > Suggested ADR directory: `<target_dir>` (based on recent file context)
      > Other options: `<list remaining dirs>`
      > Press Enter to accept, or type path of another directory.
   c. Wait for confirmation or override.
   d. Use confirmed path as `target_dir`.

## Step 3 — Determine next ADR number

1. List all files in `target_dir` matching `NNNN-*.md` (NNNN = 1+ digits).
2. Find highest number among matching filenames.
3. `next_num = highest + 1` (or `1` if no matching files exist).
4. Zero-pad `next_num` to 4 digits: e.g., `1` → `0001`, `12` → `0012`.

## Step 4 — Parse the decision context

User may provide decision as arg to `/adr-create <decision summary>`, or it may be in conversation context.

1. **Arg provided** (e.g., `/adr-create Use PostgreSQL for primary storage`): use arg text as `decision_summary`.
2. **No arg, but conversation context** contains clear architectural decision: extract from context as `decision_summary`.
3. **No arg and no clear context**: ask:
   > What architectural decision are you recording? (e.g., "Use PostgreSQL for primary storage")

From `decision_summary` and full conversation context, derive:
- `adr_title`: concise title (summary itself, or shortened form if verbose).

Convert `adr_title` to kebab-case slug for filename: lowercase, spaces and special chars → hyphens.

## Step 5 — Draft all ADR sections

Skill authors every section — do not leave template placeholders. Infer content from conversation context, codebase state, `decision_summary`. Insufficient info → use `AskUserQuestion` before proceeding.

### Context

Context: describe problem forces, not solution. Don't advocate chosen approach.

Write short paragraph explaining **why** decision needed. Draw from:
- Conversation history (what problem was being discussed?)
- Recent code changes or files under discussion
- Any constraints, requirements, or trade-offs mentioned

Info insufficient → ask:
> What problem or situation prompted this decision? What constraints are in play?

### Decision

Decision: active voice ("We decided to X" / "We will use Y"), specific, declarative. State what chosen, not why (→ Context) or what happens next (→ Consequences).

Write few sentences stating **what** was decided. Draw from:
- `decision_summary` arg
- Any explicit choices made in conversation

Decision ambiguous → ask:
> Can you state the decision more precisely? What exactly are we choosing to do?

### Consequences

Consequences: min 1 adverse outcome required; 3–7 bullets; 200–400 words. Every choice involves giving something up.

Write 3–7 bullets covering **positive** and **negative** consequences:
- Benefits expected
- Trade-offs or risks accepted (at least one — required)
- Follow-up work created
- Migration or compatibility implications, if applicable

Consequences can't be inferred → ask:
> What are the main benefits and trade-offs of this decision? Any follow-up work it creates?

### Interview flow

Batch questions — if clarity needed on multiple sections, ask together in single `AskUserQuestion` call. Proceed to file creation only after all sections have substantive content.

## Step 6 — Create the ADR file

1. Construct filename: `<target_dir>/<NNNN>-<slug>.md`.
2. Fill template from [`references/0000-adr-template.md`](references/0000-adr-template.md):
   - Replace `NNNN` with zero-padded number.
   - Replace `Title` with `adr_title`.
   - Set `**Status:** Proposed` (new ADRs always start Proposed unless user specifies otherwise).
   - Set `**Date:**` to today's date in `YYYY-MM-DD`.
   - Write drafted Context, Decision, Consequences into respective sections.
3. Write file to disk.

## Step 7 — Update the index

1. Check `<target_dir>/README.md` exists.
   - Missing → create with structure:
     ```markdown
     # Architecture Decision Records

     | ADR | Title | Status |
     |-----|-------|--------|
     ```
2. Add new row to table:
   ```
   | [ADR-NNNN](<NNNN>-<slug>.md) | <adr_title> | Proposed |
   ```
   Insert at end of table (after last existing row).
3. Save `README.md`.

## Step 8 — Confirm

Inform user:

> Created `<target_dir>/<NNNN>-<slug>.md` (ADR-NNNN: <adr_title>)
> Updated index: `<target_dir>/README.md`
>
> Review the ADR and change the Status to `Accepted` when the decision is finalised.
>
> Tip: commit this ADR in the same PR as (or just before) the code it describes so decision and implementation are traceable together.

## Step 9 — Post-write validation

1. Invoke `adr-check` in scoped mode against newly created ADR file:
   `adr-check <target_dir>/<NNNN>-<slug>.md`
2. Review result:
   - **Structural FAIL:** Block completion and present issues to user. Prompt to resolve structural problems before confirming completion. Offer to fix directly if straightforward (e.g., missing section).
   - **Style warnings:** Display to user but don't block. ADR considered complete; user may address or not.
   - **PASS (no warnings):** Skill completes successfully.
