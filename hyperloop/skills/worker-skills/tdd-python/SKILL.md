---
name: tdd-python
description: TDD discipline for Python tasks — red/green/refactor cycle, pytest patterns, uv toolchain.
user-invocable: false
---

# TDD Python

**Use:** Python impl tasks — new modules, fixes, refactors.

## Cycle

1. Write failing test first (`pytest`). Name: `test_<what>_<condition>`.
2. Implement minimum code → green.
3. Refactor → green.
4. Repeat per behaviour slice.

## Toolchain

- Run: `uv run pytest` (not bare `pytest` — respects virtualenv).
- Lint/format: `uv run pre-commit run --all-files` before commit.
- Type hints required on public API. Check: `uv run mypy <module>` if mypy configured.

## Patterns

- One assert per test concept (ok to have multiple asserts if same concept).
- Fixtures in `conftest.py` — don't repeat setup across test files.
- Mock at boundary only (external I/O, network, DB). Don't mock internal functions.
- `pytest.raises` for expected exceptions — never bare `try/except` in tests.

## Do / Don't

- DO: test behaviour, not implementation details.
- DON'T: test private methods directly — refactor if you feel the need.
- DON'T: commit with failing tests. Red → fix → green → commit.
- DON'T: use `unittest.TestCase` unless repo already does — prefer plain functions + pytest.
