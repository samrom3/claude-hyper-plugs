# adr-wizard

> ADR lifecycle management for Claude Code.

<!-- mdformat-toc start --slug=github --maxlevel=3 --minlevel=2 -->

- [Overview](#overview)
- [Installation](#installation)
- [Skills](#skills)
  - [adr-create](#adr-create)
  - [adr-supersede](#adr-supersede)
  - [adr-deprecate](#adr-deprecate)
  - [adr-check](#adr-check)
- [ADR Location Convention](#adr-location-convention)
- [Compatibility](#compatibility)

<!-- mdformat-toc end -->

## Overview<a name="overview"></a>

`adr-wizard` is a Claude Code plugin that provides focused skills for every stage of an
Architecture Decision Record (ADR) lifecycle: **create**, **supersede**, **deprecate**, and
**validate**. It supports multiple ADR directories in a single repository via a CLAUDE.md
convention, and integrates gracefully with the `hyperloop` gate when both plugins are installed.

## Installation<a name="installation"></a>

Make sure you have the [`hyper-plugs`](/README.md) marketplace installed, then install `adr-wizard`:

```bash
claude plugin install adr-wizard@hyper-plugs
```

## Skills<a name="skills"></a>

| Skill           | Command          | Description                                                    |
| --------------- | ---------------- | -------------------------------------------------------------- |
| `adr-create`    | `/adr-create`    | Create a new ADR with auto-numbering and index updates         |
| `adr-supersede` | `/adr-supersede` | Supersede an existing ADR, with bidirectional cross-references |
| `adr-deprecate` | `/adr-deprecate` | Deprecate an ADR with a recorded reason                        |
| `adr-check`     | `/adr-check`     | Validate all ADRs for structural integrity and index sync      |

### adr-create<a name="adr-create"></a>

Creates a new Nygard-style ADR file with auto-numbering, fills in the template, and updates the
directory index.

```
/adr-create
# or: just discuss an architectural decision in the conversation and the
# model will offer to create an ADR automatically
```

The skill discovers your ADR directory from CLAUDE.md, finds the next available number, creates
`NNNN-<slug>.md`, and adds it to `README.md`. It drafts all sections (Context, Decision,
Consequences) from conversation context, interviewing you for any missing details.

### adr-supersede<a name="adr-supersede"></a>

Creates a new ADR that replaces an existing one. Updates the old ADR's status to
`Superseded by ADR-NNNN` and adds a `Supersedes: ADR-MMMM` field to the new one.

```
/adr-supersede
# optionally supply the ADR number to supersede:
/adr-supersede 0003
```

### adr-deprecate<a name="adr-deprecate"></a>

Marks an existing ADR as deprecated without creating a replacement. Records the deprecation
reason in the file.

```
/adr-deprecate
# optionally supply the ADR number:
/adr-deprecate 0005
```

### adr-check<a name="adr-check"></a>

Validates all ADR directories for structural integrity, index sync, and cross-reference
consistency. Also scans `git diff` for patterns that may indicate undocumented architectural
decisions (advisory warnings only).

```
/adr-check
```

Exit result: **PASS** (all checks clear) or **FAIL** (with a remediation report listing exactly
what needs fixing and in which files).

## ADR Location Convention<a name="adr-location-convention"></a>

`adr-wizard` skills discover ADR directories by reading a `### ADR Locations` subsection from
the project's `CLAUDE.md`. Add this subsection wherever it makes sense in your `CLAUDE.md`:

```markdown
### ADR Locations
- docs/adrs/           # project-wide decisions
- hyperloop/docs/adrs/ # hyperloop plugin decisions
```

Each bullet is a relative path to an ADR directory. Inline `# comments` are ignored by the
skills.

If no `### ADR Locations` section is present, skills fall back to scanning for `docs/adrs/`,
`decisions/`, and `architecture/decisions/` at the repository root.

## Compatibility<a name="compatibility"></a>

`adr-wizard` works standalone — no other plugins required. When used alongside `hyperloop`,
the gate template automatically delegates ADR validation to `/adr-check` if the skill is
available, providing multi-directory ADR support in the gate. If `adr-wizard` is not installed,
`hyperloop` falls back to its built-in basic ADR scanning.
