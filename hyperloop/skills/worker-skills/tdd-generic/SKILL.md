---
name: tdd-generic
description: Generic TDD discipline — fallback for tasks where language/stack is unspecified or mixed. Red/green/refactor cycle adapted to available tooling.
user-invocable: false
---

# TDD Generic

**Use:** Fallback skill for tasks with unspecified language/stack, or tasks spanning multiple languages. Use `tdd-python` or `tdd-typescript` instead when stack is known.

## Cycle

1. Write failing test first — define expected behaviour before code.
2. Implement minimum code → test passes.
3. Refactor → still passes.
4. Repeat per behaviour slice.

## Adapting to Unknown Stack

1. Read `CLAUDE.md` and `package.json` / `pyproject.toml` / `Makefile` / `Cargo.toml` — find test command.
2. Run existing tests to confirm baseline is green before touching anything.
3. Follow existing test file naming and structure exactly.
4. If no test framework present: note the gap in PR description, write tests closest to what the project would support.

## Verification Before Commit

- Run whatever the project's lint+test command is (`pre-commit run --all-files`, `make test`, `npm test`, etc.).
- Must be green. No exceptions.

## Do / Don't

- DO: use this skill only when stack is genuinely unknown — prefer a specific skill when possible.
- DO: search for existing test files before writing any test — understand conventions first.
- DON'T: invent a new testing framework — use what's already in the repo.
- DON'T: commit code without tests unless task explicitly says docs/config only.
- DON'T: skip verification because the stack is unfamiliar — investigate until green.
