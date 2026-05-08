---
name: tdd-typescript
description: TDD discipline for TypeScript tasks — red/green/refactor, vitest/ts-jest, tsc type checking, tsconfig awareness.
user-invocable: false
---

# TDD TypeScript

**Use:** TypeScript impl tasks — new modules, fixes, refactors.

## Cycle

1. Write failing test first. Name: `<what>.<condition>.test.ts` or `<what>.spec.ts`.
2. Implement minimum code → green.
3. Refactor → green.
4. Repeat per behaviour slice.

## Toolchain

- Test runner: `vitest` preferred; `ts-jest` if repo uses Jest. Check `package.json` scripts first.
- Type check: `tsc --noEmit` (never skip — catches type errors tests miss).
- Lint: `eslint` + `prettier` per repo config. Run before commit.
- `tsconfig.json`: read it. Match `target`, `moduleResolution`, `strict` settings in new code.

## Patterns

- Use `describe` / `it` blocks. Name behaviour: `it('returns null when input is empty')`.
- Typed mocks: `vi.fn<Args, Return>()` (vitest) or `jest.fn()` typed via generics.
- `expect.assertions(N)` in async tests — prevents silent pass on missed await.
- Fixtures as typed constants in `__fixtures__/` or co-located `*.fixtures.ts`.

## Do / Don't

- DO: run `tsc --noEmit` before declaring green — type errors are bugs.
- DO: check `tsconfig.json` paths/aliases before adding imports — broken at emit.
- DON'T: use `any` to silence type errors — fix the type.
- DON'T: commit with `@ts-ignore` without a comment explaining why.
- DON'T: mock modules not at a system boundary — test the real code.
