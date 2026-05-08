# ADR-003: Session-Spec Metadata Table Contract

**Status:** Accepted; Supersedes ADR-002

**Date:** 2026-05-07

______________________________________________________________________

## Context

ADR-002 established the two-location `source_issues` storage contract. Two contract parties changed: the writer skill is now `/session-spec` (was `prd`) and the output file is now `plans/<branch>-session-spec.md` (was the prd variant). The first `##` section heading also changed to `## Goal` (was `## 1. Introduction/Overview`).

The decision and contract wording are unchanged — only the contract parties (skill name, file name, first section heading) are updated here.

______________________________________________________________________

## Decision

Store `source_issues` in **both** locations, with a clear authority hierarchy:

| Location                                   | Role                                                     | Mutability                      |
| ------------------------------------------ | -------------------------------------------------------- | ------------------------------- |
| Spec metadata table                        | Human-visible record; written once by `/session-spec`    | Immutable after spec is written |
| `team-state.json` `metadata.source_issues` | Authoritative runtime value; read by Phase 1 and Phase 4 | Immutable after first write     |

**Write path:** `/session-spec` (Step 1) detects all issue URLs in `$ARGUMENTS`, writes one `| Source Issue |`
row per issue in the spec metadata table, and assigns each issue. Phase 1 collects all rows and
copies the list into `team-state.json` as `source_issues`.

**Read path:** All agents (Phase 1 assignment verification, Phase 4 PR creation) read
`metadata.source_issues` from `team-state.json`. No agent re-parses the spec after Phase 1.

### Canonical metadata table format

The metadata table format is a binding contract between `/session-spec` (writer) and Phase 1 (reader).
It must be followed exactly:

```markdown
# <Title>

| Field        | Value                         |
| ------------ | ----------------------------- |
| Source Issue | owner/repo#N                  |
| Source Issue | owner/repo#M                  |

## Goal
```

For a single issue, the table contains exactly one `Source Issue` row.

**Placement rules:**

- The table appears **immediately after the H1 heading** (one blank line separating them).
- The table appears **before the first `##` section heading**.
- If no issue URL was provided, the table is **omitted entirely** — the H1 heading is followed
  directly by `## Goal`.

**Parsing contract for agents:**

1. Locate the H1 heading (`# ...`).
1. Scan forward line by line.
1. Collect **all** `| Source Issue |` rows found **before** the first `##` line; extract the
   second cell value from each row and build the `source_issues` list.
1. If a `##` line is reached before any `| Source Issue |` row, `source_issues` is `null`.

**Immutability:** `source_issues` MUST NOT be mutated after `team-state.json` is first written.

______________________________________________________________________

## Consequences

Same as ADR-002. The rename introduces no new trade-offs.
