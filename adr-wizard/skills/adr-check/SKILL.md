---
name: adr-check
description: "This skill should be used when the user asks to 'validate ADRs', 'check ADR structure', 'run adr-check', 'check for undocumented decisions', or when lifecycle skills and gate agents need to validate ADR files. Supports Global mode (all discovered ADR directories, including diff-based warnings), Scoped mode (single file, directory, or natural-language query — no diff noise), and future Diff mode."
user-invocable: true
argument-hint: "[optional: path/to/NNNN-adr-file.md | path/to/adrs/ | natural language query]"
---

# adr-check

Validates ADRs against `references/adr-check-contract.md`. Modes: **Global** (all ADR dirs + diff warnings), **Scoped** (file/dir/query — no diff noise), **Diff** (changed ADRs on branch). Outputs structured pass/fail report.

---

## Step 1 — Determine mode and discover inputs

1. Inspect skill arg:
   - **No arg** → `mode = global`. Run dir discovery (steps 2–6).
   - **Arg = `.md` file** → `mode = scoped_file`, `target = <path>`. Skip dir discovery. Checks 2.2, 2.3, Step 4 skipped.
   - **Arg = directory** → `mode = scoped_dir`, `target = <directory>`. Skip CLAUDE.md discovery. Use `target` as sole `adr_dirs` entry. Run Checks 2.1, 2.1e, 2.2, 2.3. Skip Step 4.
   - **Any other non-empty arg** → `mode = scoped_query`, `query = <arg>`. Run dir discovery. Match ADR files via model judgment. Run Checks 2.1 and 2.1e only. Skip 2.2, 2.3, Step 4. No matches → emit advisory warning, exit PASS.

2. *(global/scoped_query)* Read project `CLAUDE.md`.
3. *(global/scoped_query)* Search for heading containing `ADR Locations` (case-insensitive).
4. *(global/scoped_query)* If found, collect bullet items as relative paths, strip inline `# comments`. Store as `adr_dirs`.
5. *(global/scoped_query)* If not found, fall back: scan repo root for `docs/adrs/`, `decisions/`, `architecture/decisions/`.
6. *(global only)* `adr_dirs` empty → output report `Mode: Global`, emit `WARNING: No ADR directories found. Skipping validation.`, set `Overall: PASS`, stop.

## Step 2 — Validate each directory

For each dir in `adr_dirs`, run checks below. Track failures as `issues`.

### Check 2.1 — ADR file structure

1. List files matching `NNNN-*.md` (1+ digits, hyphen, any chars, `.md`).
2. For each ADR file, verify:
   a. **Status present/non-empty:** Line matching `**Status:**` or `## Status` with non-empty value. Else: `[ADR-NNNN] Missing or empty Status field`
   b. **Context non-trivial:** `## Context` body has ≥1 non-blank non-placeholder line (not solely whitespace, `TODO`, `TBD`, or template guidance). Else: `[ADR-NNNN] ## Context section is empty or contains only placeholder text`
   c. **Decision non-trivial:** Same for `## Decision`. Else: `[ADR-NNNN] ## Decision section is empty or contains only placeholder text`
   d. **Consequences present:** `## Consequences` section exists (content may be brief). Else: `[ADR-NNNN] ## Consequences section is missing`
   e. **Consequences quality (style):** Model judgment — ≥1 genuinely adverse outcome (risk, trade-off, migration cost, complexity increase, regression, reduced flexibility, or explicit statement of what's given up). If absent, **style warning** (not structural): `[ADR-NNNN] Consequences section contains no clearly adverse consequence or trade-off — consider adding one for credibility`. Style warnings don't affect pass/fail.

### Check 2.2 — Index sync *(whole-dir mode only)*

1. If `<dir>/README.md` missing: issue `README.md index is missing from <dir>`. Skip index checks for this dir.
2. Parse README.md table: extract all ADR filenames/numbers linked in table.
3. Each ADR file w/o table entry → issue: `[ADR-NNNN] ADR file exists but has no entry in README.md index`
4. Each table entry w/o matching file → issue: `[index entry] README.md references <filename> but file does not exist (orphaned entry)`

### Check 2.3 — Cross-reference integrity *(whole-dir mode only)*

1. ADR with status `Superseded by ADR-NNNN`:
   a. Verify `NNNN-*.md` exists. Else: `[ADR-MMMM] Status says "Superseded by ADR-NNNN" but ADR-NNNN does not exist`
   b. Verify referenced ADR contains `Supersedes: ADR-MMMM`. Else: `[ADR-NNNN] Missing "Supersedes: ADR-MMMM" back-reference`
2. ADR with `Supersedes: ADR-NNNN`:
   a. Verify `NNNN-*.md` exists. Else: `[ADR-MMMM] References "Supersedes: ADR-NNNN" but ADR-NNNN does not exist`
   b. Verify ADR-NNNN status is `Superseded by ADR-MMMM`. Else: `[ADR-NNNN] Expected status "Superseded by ADR-MMMM" but found different status`

## Step 3 — Build the validation report

Separate Check 2.1e items into `style_warnings` (not structural, don't affect status).

Per validated target: structural issues (2.1a–2.1d) present → FAIL, else PASS.

Output report (header rows **always present**):

```
ADR Check Report
================

Mode:    Global | Scoped (file) | Scoped (directory) | Scoped (query)
Target:  <path, directory, or natural-language query>
ADRs:    <comma-separated list of all validated ADR filenames>

Results
-------
<path/to/directory-or-file>:
  Status: PASS | FAIL
  Issues:
    - [ADR-NNNN] <issue description>

<path/to/next-item>:
  Status: PASS
  Issues: none

Style Warnings (advisory only — not gate-blocking)
===================================================
  - [ADR-NNNN] <style warning message>

Overall: PASS | FAIL
```

Global/scoped_dir → group by dir. scoped_file/scoped_query → each file own entry.

`Style Warnings` section omitted if empty.

Any entry FAIL → `Overall: FAIL`. On FAIL, append `Remediation` section (each issue + fix action).

## Step 4 — Diff-based warnings *(Global only — skip all scoped sub-modes)*

1. Run `git diff HEAD`.
2. Scan for:

   | Pattern | Warning |
   |---------|---------|
   | `+` lines defining new ABC/interface (Python `class Foo(ABC)`, `abstractmethod`, Java/TS/Go `interface`, TS `abstract class`) | "New interface/ABC detected in `<file>` — consider documenting the design decision with `/adr-create`" |
   | `+` lines in new file matching `*Config*`, `*Settings*`, `*Configuration*` (case-insensitive) | "New configuration file `<file>` — consider documenting configuration decisions with `/adr-create`" |
   | `+` lines in `package.json` (deps/devDeps), `pyproject.toml` ([tool.poetry.dependencies]/[project]), `Cargo.toml` ([dependencies]), `go.mod` (require) | "New dependency added in `<file>` — consider documenting the technology choice with `/adr-create`" |
   | New root/top-level dir named `service`, `module`, `component`, `pkg`, `lib`, `api`, or `gateway` | "New service/module boundary `<dir>` — consider documenting the architectural boundary with `/adr-create`" |

3. Warnings found → append to report:
   ```
   Diff-Based Warnings (advisory only — not gate-blocking)
   ========================================================
     - [WARNING] <message>
   ```
   No warnings → omit section.

## Step 5 — Output and exit

Print full report.

- **PASS** (with/without diff/style warnings): success. Gate consumers continue.
- **FAIL**: failure. Gate consumers block; present remediation to user.

## Additional Resources

- **`references/adr-check-contract.md`** — Full contract: discovery rules, checks (2.1–2.3, 2.1e), report format, pass/fail semantics, invocation modes.
