---
name: hyperwork-python
description: Python tooling for impl tasks — pytest patterns, uv toolchain, mypy, fixtures. Compose with hyperwork-tdd skill for methodology.
user-invocable: false
---

# Python

**Use:** Python impl tasks. Load with `hyperwork-tdd` skill: `skills: [hyperwork-tdd, hyperwork-python]`.

## Toolchain

- Run tests: `uv run pytest` (not bare `pytest`).
- Lint/format: `uv run pre-commit run --all-files` before commit.
- Types: `uv run mypy <module>` if mypy configured. Required on public API.

## Patterns

- Test name: `test_<what>_<condition>`.
- One assert per concept (multiple asserts OK if same concept).
- Fixtures in `conftest.py` — no repeated setup across test files.
- Mock at boundary only (external I/O, network, DB). Don't mock internals.
- `pytest.raises` for expected exceptions — never bare `try/except` in tests.

## Do / Don't

- DON'T: use `unittest.TestCase` unless repo already does — prefer plain functions + pytest.
- DON'T: test private methods directly — refactor if tempted.
- DON'T: use bare `pytest` — always `uv run pytest`.
