# ADR-Check Contract

Defines generic contract `adr-check` implements and gate agents consume. Convention-based — no runtime dependency between plugins. Any skill satisfying this interface can serve as ADR validator for a gate.

---

## 1. Discovery — Inputs

### 1.1 Whole-Directory Mode (default)

Skill MUST discover ADR dirs using this priority order:

1. **CLAUDE.md convention (primary):** Search project `CLAUDE.md` for heading containing `ADR Locations` (case-insensitive, any level). Under that heading, read each bullet item as relative path to ADR dir. Inline `# comment` annotations ignored. Example:

   ```markdown
   ### ADR Locations
   - docs/adrs/           # project-wide decisions
   - hyperloop/docs/adrs/ # hyperloop plugin decisions
   ```

2. **Fallback (no ADR Locations heading):** Scan repo root for `docs/adrs/`, `decisions/`, `architecture/decisions/`.

No ADR dirs found → skill MUST report as **warning** (not failure) and exit passing. Project with no ADR dirs has nothing to validate.

### 1.2 Scoped Mode

Invoked with arg → skill enters **scoped mode**. Three sub-modes:

- **Scoped (file):** Arg is path to single `.md` file (e.g., `/adr-check docs/adrs/0001-foo.md`). Validates only that file. Runs Section 2.1 and 2.1e only. Sections 2.2, 2.3, 4 skipped.

- **Scoped (directory):** Arg is dir path (e.g., `/adr-check docs/adrs/`). Validates all ADR files in that dir. Runs Sections 2.1, 2.1e, 2.2, 2.3 for that dir only. Section 4 skipped.

- **Scoped (query):** Arg is natural-language description (e.g., `/adr-check "validate only ADRs impacting the Zip Event management components"`). Skill uses model judgment to identify matching ADR files across all discovered dirs. Runs Section 2.1 and 2.1e against each matched file. Sections 2.2, 2.3, 4 skipped.

All scoped sub-modes skip Section 4. Section 4 is exclusive to Global mode.

Scoped mode for targeted validation: lifecycle skills use `Scoped (file)` for post-write validation; users/tools use `Scoped (directory)` or `Scoped (query)` to narrow check without diff-based noise.

---

## 2. Validation — What Is Checked

For each discovered ADR dir, skill runs these checks:

### 2.1 ADR File Structure

Each file matching `NNNN-*.md` in dir is ADR file. For each, verify:

- File contains `## Status` section (or equivalent heading) with non-empty value.
- File contains `## Context` with non-trivial content (more than one blank line or placeholder text like "TODO").
- File contains `## Decision` with non-trivial content.
- File contains `## Consequences` (may be brief; existence sufficient).

### 2.1e Style Check — Consequences Quality

Fires on every validated ADR file (both whole-directory and scoped mode).

Skill uses model judgment — not keyword matching — to assess whether `## Consequences` contains at least one genuinely adverse outcome. Qualifying examples: known risk, trade-off, migration cost, complexity increase, performance regression, reduced flexibility, or explicit statement of what is given up.

None found → skill emits **style warning** (not structural failure):

```
[ADR-NNNN] Consequences section contains no clearly adverse consequence or trade-off — consider adding one for credibility
```

Style warnings are informational. Don't affect pass/fail result.

### 2.2 Index Sync

ADR dir MUST contain `README.md` as index. Verify:

- Every ADR file has corresponding entry in README.md index.
- Every README.md index entry has corresponding ADR file. (No orphaned entries pointing to missing files.)

### 2.3 Cross-Reference Integrity

ADRs with `Superseded by ADR-NNNN` status:
- Referenced ADR (NNNN) must exist in dir.
- Referenced ADR must contain `Supersedes: ADR-MMMM` back-reference.

ADRs with `Supersedes: ADR-NNNN` field:
- Referenced ADR (NNNN) must exist.
- Referenced ADR's status must be `Superseded by ADR-<this>`.

---

## 3. Output — Validation Report

Skill MUST output structured report. Format identical regardless of mode; `Mode` and `ADRs` header rows provide context.

```
ADR Check Report
================

Mode:    Global | Scoped (file) | Scoped (directory) | Scoped (query)
Target:  <path, directory path, or natural-language query>
ADRs:    <comma-separated list of all ADR filenames validated, e.g. "0001-foo.md, 0002-bar.md">

Results
-------
<path/to/directory-or-file>:
  Status: PASS | FAIL
  Issues:
    - [ADR-NNNN] <description of issue>

<path/to/next-item>:
  Status: PASS
  Issues: none

Style Warnings (advisory only — not gate-blocking)
===================================================
  - [ADR-NNNN] <style warning message>

Diff-Based Warnings (advisory only — not gate-blocking)
========================================================
  - [WARNING] <message>

Overall: PASS | FAIL
```

**Results grouping:**
- **Global** and **Scoped (directory):** Group by dir path.
- **Scoped (file)** and **Scoped (query):** Each validated file as own result entry.

**Conditional sections:**
- `Style Warnings` — appears only when Section 2.1e emits ≥1 warning; omitted otherwise.
- `Diff-Based Warnings` — appears only in Global mode when Section 4 emits warnings; omitted in all scoped sub-modes and when no warnings found.

On failure, each issue entry MUST include:
- Which ADR file affected (by number and filename)
- What check failed
- Specific remediation step (e.g., "Add a non-empty ## Context section to docs/adrs/003-foo.md")

---

## 4. Diff-Based Warnings

After structural validation, skill MUST scan `git diff HEAD` (staged + unstaged) for patterns suggesting undocumented architectural decisions. Surfaced as **warnings** — do NOT affect pass/fail, do NOT block gate.

Patterns to detect:

| Pattern | Warning message |
|---------|----------------|
| New abstract class or interface definition (Python ABC, Java `interface`, Go `interface{}` with multiple methods, TypeScript `interface`/`abstract class`) | "New interface/ABC detected in \<file\> — consider documenting the design decision" |
| New file matching `*Config*`, `*Settings*`, `*Configuration*` | "New configuration file detected — consider documenting configuration decisions" |
| Addition of new dependency in `package.json`, `pyproject.toml`, `Cargo.toml`, or `go.mod` | "New dependency added — consider documenting the technology choice" |
| New top-level dir or new subdir named `service`, `module`, `component`, `pkg`, `lib`, `api`, or `gateway` | "New service/module boundary detected — consider documenting the architectural boundary" |

Warnings appended to report after main validation output:

```
Diff-Based Warnings (advisory only — not gate-blocking)
========================================================
  - [WARNING] <pattern description>: <file or location>
```

No warnings found → section omitted from report.

---

## 5. Pass / Fail Semantics

| Mode | Condition | Result |
|------|-----------|--------|
| Global | All structural checks pass | **PASS** |
| Global | Any structural check fails | **FAIL** |
| Global | No ADR directories found | **PASS** (warning emitted) |
| Global | Diff-based warnings present | **PASS** (informational only) |
| Global | Style warnings present | **PASS** (informational only) |
| Scoped (any) | Structural check fails on target | **FAIL** |
| Scoped (any) | Style warning emitted | **PASS** (informational only) |
| Scoped (any) | Diff-based warnings | Skipped — not evaluated |
| Scoped (query) | No ADRs match the query | **PASS** (advisory warning emitted) |

Gate consumers MUST treat **FAIL** as gate-blocking. Gate consumers MUST treat diff-based warnings as informational only — log in progress file and present to user, but don't block gate.

---

## 6. Invocation

- **Global mode:** `/adr-check`
  No arg. Validates all ADR dirs discovered via Section 1.1. Runs all checks (Sections 2.1–2.3 and Section 4).

- **Scoped (file):** `/adr-check path/to/NNNN-adr-file.md`
  Arg is `.md` file path. Validates only that file. Runs Sections 2.1 and 2.1e; skips 2.2, 2.3, 4.

- **Scoped (directory):** `/adr-check path/to/adrs/`
  Arg is dir path. Validates all ADR files in that dir. Runs Sections 2.1, 2.1e, 2.2, 2.3; skips Section 4.

- **Scoped (query):** `/adr-check "natural language description"`
  Arg is quoted or unquoted natural-language string (not a resolvable path). Model identifies matching ADRs, runs Sections 2.1 and 2.1e; skips 2.2, 2.3, 4.

- **Lifecycle skill invocation:** Lifecycle skills (`adr-create`, `adr-supersede`, `adr-deprecate`) invoke `Scoped (file)` against newly written file. Validates only that ADR without noise from other files or diff patterns.

- **Gate invocation:** Gate template calls skill in Global mode. If skill not installed, gate falls back to basic dir scanning (see FR-011 in PRD).
