---
name: hyperwork-tdd
description: Language-agnostic TDD discipline — red/green/refactor cycle. Fallback for unknown stack. Compose with language skills (hyperwork-python, hyperwork-typescript) for tooling specifics.
user-invocable: false
---

# TDD

**Use:** Any impl task. Compose with language skill for tooling. Standalone = fallback.

## Cycle

1. Write failing test first — defines expected behaviour.
2. Implement minimum code → test passes.
3. Refactor → still passes.
4. Repeat per behaviour slice.

## Adapting to Unknown Stack

1. Read `CLAUDE.md` + `package.json` / `pyproject.toml` / `Makefile` / `Cargo.toml` → find test command.
2. Run existing tests first — confirm baseline green before touching anything.
3. Match existing test file naming and structure exactly.
4. No test framework present → note gap in PR description; write closest supported equivalent.

## Verification Before Commit

Run project lint+test command (`pre-commit run --all-files`, `make test`, `npm test`, etc.). Must be green. No exceptions.

## Do / Don't

- DO: test behaviour, not implementation details.
- DO: search existing tests before writing any — understand conventions first.
- DON'T: commit with failing tests.
- DON'T: invent new testing framework — use what's in the repo.
- DON'T: skip verification because stack is unfamiliar — investigate until green.
