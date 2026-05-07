---
name: adr-check
description: "This skill should be used when the user asks to 'validate ADRs', 'check ADR structure', 'run adr-check', 'check for undocumented decisions', or when lifecycle skills and gate agents need to validate ADR files. Supports Global mode (all discovered ADR directories, including diff-based warnings), Scoped mode (single file, directory, or natural-language query — no diff noise), and future Diff mode."
user-invocable: true
argument-hint: "[optional: path/to/NNNN-adr-file.md | path/to/adrs/ | natural language query]"
---

# adr-check

Validates ADRs against contract in `references/adr-check-contract.md`. Three modes:

* **Global** — all discovered ADR dirs, incl. diff-based warnings
* **Scoped** — single file, dir, or query; no diff noise
* **Diff** — ADRs changed in current branch

Outputs structured pass/fail report with consistent header regardless of mode.

---

## Step 1 — Determine mode and discover inputs

1. Inspect skill arg:

   - **No arg** → `mode = global`. Continue with dir discovery (steps 2–6).

   - **Arg resolves to `.md` file** → `mode = scoped_file`, `target = <path>`.
     Skip dir discovery. Proceed to Step 2 with only that file.
     Checks 2.2, 2.3, Step 4 skipped entirely.

   - **Arg resolves to dir** → `mode = scoped_dir`, `target = <directory>`.
     Skip CLAUDE.md discovery. Use `target` as sole entry in `adr_dirs`.
     Run Checks 2.1, 2.1e, 2.2, 2.3 for that dir. Skip Step 4.

   - **Any other non-empty arg** → `mode = scoped_query`, `query = <argument>`.
     Run dir discovery (steps 2–6). Use model judgment to identify ADR files matching query.
     Run only Checks 2.1 and 2.1e against matched files. Skip 2.2, 2.3, Step 4.
     No ADRs match → emit advisory warning and exit PASS.

2. *(global and scoped_query only)* Read project `CLAUDE.md`.
3. *(global and scoped_query only)* Search for heading containing `ADR Locations` (case-insensitive, any level).
4. *(global and scoped_query only)* Found → collect each bullet item as relative path, strip inline `# comments`. Store as `adr_dirs`.
5. *(global and scoped_query only)* Not found → fall back: scan repo root for `docs/adrs/`, `decisions/`, `architecture/decisions/`.
6. *(global only)* `adr_dirs` empty after both methods → output report `Mode: Global`, emit: `WARNING: No ADR directories found. Skipping validation.` Set `Overall: PASS` and stop.

## Step 2 — Validate each directory

For each dir in `adr_dirs`, run all checks below. Track failures as `issues` list.

### Check 2.1 — ADR file structure

1. List all files matching `NNNN-*.md` (1+ digits, hyphen, any chars, `.md`).
2. For each ADR file, verify:
   a. **Status present and non-empty:** File contains `**Status:**` (or `## Status`) with non-empty value. Missing/empty → issue: `[ADR-NNNN] Missing or empty Status field`
   b. **Context non-trivial:** `## Context` body has 1+ non-blank non-placeholder lines. Missing/trivial → issue: `[ADR-NNNN] ## Context section is empty or contains only placeholder text`
   c. **Decision non-trivial:** Same check for `## Decision`. Failing → issue: `[ADR-NNNN] ## Decision section is empty or contains only placeholder text`
   d. **Consequences present:** File contains `## Consequences`. Entirely absent → issue: `[ADR-NNNN] ## Consequences section is missing`
   e. **Consequences quality (style check):** Model judgment — not keyword matching — assess whether `## Consequences` contains at least one genuinely adverse outcome (risk, trade-off, migration cost, complexity increase, perf regression, reduced flexibility, or explicit statement of what is given up). None found → **style warning** (not structural issue): `[ADR-NNNN] Consequences section contains no clearly adverse consequence or trade-off — consider adding one for credibility`
      Style warnings don't affect pass/fail; reported separately (see Step 3).

### Check 2.2 — Index sync *(whole-directory mode only)*

1. Check `<dir>/README.md` exists.
   - Missing → issue: `README.md index is missing from <dir>`. Skip index sync.
2. Parse README.md table: extract all ADR filenames/numbers linked.
3. Each ADR file in dir must have corresponding table entry. Missing → issue: `[ADR-NNNN] ADR file exists but has no entry in README.md index`
4. Each README.md table entry must have corresponding file. Missing → issue: `[index entry] README.md references <filename> but file does not exist (orphaned entry)`

### Check 2.3 — Cross-reference integrity *(whole-directory mode only)*

1. For each ADR with status `Superseded by ADR-NNNN`:
   a. Verify `NNNN-*.md` exists. Missing → issue: `[ADR-MMMM] Status says "Superseded by ADR-NNNN" but ADR-NNNN does not exist`
   b. Read referenced ADR (NNNN). Verify contains `Supersedes: ADR-MMMM`. Missing → issue: `[ADR-NNNN] Missing "Supersedes: ADR-MMMM" back-reference`
2. For each ADR with `Supersedes: ADR-NNNN` field:
   a. Verify `NNNN-*.md` exists. Missing → issue: `[ADR-MMMM] References "Supersedes: ADR-NNNN" but ADR-NNNN does not exist`
   b. Read ADR NNNN. Verify status is `Superseded by ADR-MMMM`. Mismatch → issue: `[ADR-NNNN] Expected status "Superseded by ADR-MMMM" but found different status`

## Step 3 — Build the validation report

Separate Check 2.1e items into `style_warnings` list. NOT counted as structural issues, NOT affecting status.

For each validated target (dir or file):
- Structural issues (2.1a–2.1d) present → status = FAIL
- Otherwise → status = PASS

Output report using structure below. Header rows (`Mode`, `Target`, `ADRs`) **always present** regardless of mode.

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

**Results grouping:** `global`/`scoped_dir` → group by dir path. `scoped_file`/`scoped_query` → each file as own result entry.

`Style Warnings` section appears only when `style_warnings` non-empty; omit otherwise.

Any result entry FAIL → `Overall` FAIL. Otherwise PASS.

On FAIL, append `Remediation` section listing each issue with file and action needed.

## Step 4 — Diff-based warnings *(Global mode only — skip in all scoped sub-modes)*

1. Run `git diff HEAD` to capture staged + unstaged changes.
2. Scan diff for these patterns:

   | Pattern | Warning |
   |---------|---------|
   | Lines starting with `+` defining new abstract class or interface (Python `class Foo(ABC)`, `abstractmethod`, Java/TypeScript/Go `interface`, TypeScript `abstract class`) | "New interface/ABC detected in `<file>` — consider documenting the design decision with `/adr-create`" |
   | Lines starting with `+` in new file matching `*Config*`, `*Settings*`, `*Configuration*` (case-insensitive) | "New configuration file `<file>` — consider documenting configuration decisions with `/adr-create`" |
   | Lines starting with `+` in `package.json` (`dependencies`/`devDependencies`), `pyproject.toml` (`[tool.poetry.dependencies]` or `[project]`), `Cargo.toml` (`[dependencies]`), or `go.mod` (`require`) | "New dependency added in `<file>` — consider documenting the technology choice with `/adr-create`" |
   | New dir at root or top-level named `service`, `module`, `component`, `pkg`, `lib`, `api`, or `gateway` | "New service/module boundary `<dir>` — consider documenting the architectural boundary with `/adr-create`" |

3. Warnings found → append to report:

   ```
   Diff-Based Warnings (advisory only — not gate-blocking)
   ========================================================
     - [WARNING] <message>
   ```

   No warnings → omit section entirely.

## Step 5 — Output and exit

Print full report. Overall result determines exit signal to callers:

- **PASS** (with or without warnings): success. Gate consumers continue.
- **FAIL**: failure. Gate consumers block and present remediation to user.

## Additional Resources

- **`references/adr-check-contract.md`** — Full contract: discovery rules, validation checks (2.1–2.3, 2.1e), report format, pass/fail semantics, invocation modes.
