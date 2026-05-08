# Back-Pressure Gate Template

Reusable template for gate check at end of hyperteam run. Gate agent reads `plans/<branch>-team-state.json` and `plans/<branch>-progress.txt` to verify all tasks passed validation. Substitute `<slug>` and `<branch>` with actual values.

---

## Gate Agent Instructions

```
You are the back-pressure gate agent. Perform all five checks below IN ORDER.

## Check 1 — Documentation–code alignment

Verify all docs/, README.md, CONTRIBUTING.md content matches implemented code.

- Mismatched AND spec makes correct one clear → fix out-of-sync artifact.
- Mismatched AND spec ambiguous → ask user to resolve. Append resolution to new "Implementation Conflict Resolutions" section at bottom of spec file (plans/<branch>-session-spec.md). Do not modify any other spec section.

## Check 2 — ADR sync

Verify all applicable design choices made during implementation are documented as ADRs with correct Status fields (Accepted, Rejected, Superseded, etc.).

**Primary path (adr-wizard installed):** If `/adr-check` skill available, invoke it. Use pass/fail directly. FAIL → fix reported issues, re-run `/adr-check` until passes.

**Fallback path (adr-wizard not installed):** Locate ADR dirs manually:
1. Read project `CLAUDE.md` for heading containing `ADR Locations` (any level, case-insensitive). Each bullet → relative path to ADR dir.
2. If no such heading, scan repo root for `docs/adrs/`, `decisions/`, `architecture/decisions/`.
3. Per dir, verify all ADR files have non-empty Status fields and README.md index is in sync. Out of sync → update ADRs, re-validate.

No ADR dirs found via either method → check passes with warning.

## Check 3 — Pre-commit checks

Run project's verification command per CLAUDE.md. Includes lint, format, tests. Must exit 0.

## Check 4 — Acceptance criteria

Verify every acceptance criterion in each step in the session-spec is met.

## Check 5 — Success metrics

Verify every success metric in the session-spec's Success Metrics section is met.

## Progress file logging

After every check (pass or fail) and every user interaction, append to plans/<branch>-progress.txt:

## [Date/Time] - GATE-<slug>-NN
- Checks passed: [list]
- Checks failed: [list with details]
- User decisions: [any questions asked and how the user responded]
- Remediation tasks written: [list of new task IDs added to team-state.json, or "none"]
- Next gate: [GATE-<slug>-NN+1 if scheduled, or "N/A — all checks passed"]
---

## Failure escalation

Checks 1–2 fail → fix in-place as described above.

Checks 3–5 fail:

1. ITERATION GUARD: If gate_iterations in team-state.json is 4 or higher, ask user before proceeding. Message must include:
   - Current gate iteration number.
   - Summary of which checks failing and whether same checks failed repeatedly across prior iterations (recurring) or are new — read plans/<branch>-progress.txt to determine.
   - What problems remain and what remediation entries would be written if user approves.
   - Clear question: proceed with escalation, or user intervenes directly?
   Do not proceed with steps 2–3 until user responds. Guard applies every gate iteration from 4th onward.

2. Write remediation entries to team-state.json (new task objects with status: pending). Append summary of each to plans/<branch>-progress.txt.

3. Increment gate_iterations in team-state.json.

4. Signal team lead via SendMessage:

   > GATE FAIL — <summary of which checks failed>
   > Remediation tasks written to team-state.json. Please re-seed native tasks.

   Do NOT call TaskCreate — lead is responsible for seeding native task list from remediation entries.
```

---

## Gate entry format reminder

- **Gate iterations:** tracked via `gate_iterations` integer in `plans/<branch>-team-state.json`.
- **Numbering:** Each successive gate increments `gate_iterations` by 1.
- **Remediation tasks:** new objects in `tasks` array with `status: pending`, appropriate `role_hint`, appropriate `blocked_by`.
- **Native task seeding:** lead calls `TaskCreate` per remediation task after receiving GATE FAIL signal — gate agent does not create native tasks directly.
