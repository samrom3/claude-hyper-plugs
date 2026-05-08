---
name: hyperwork-typescript
description: TypeScript tooling for impl tasks — vitest/ts-jest, tsc type checking, tsconfig awareness. Compose with hyperwork-tdd skill for methodology.
user-invocable: false
---

# TypeScript

**Use:** TypeScript impl tasks. Load with `hyperwork-tdd` skill: `skills: [hyperwork-tdd, hyperwork-typescript]`.

## Toolchain

- Test runner: `vitest` preferred; `ts-jest` if repo uses Jest. Check `package.json` scripts first.
- Type check: `tsc --noEmit` (never skip — catches type errors tests miss).
- Lint: `eslint` + `prettier` per repo config. Run before commit.
- `tsconfig.json`: read it. Match `target`, `moduleResolution`, `strict` in new code.

## Patterns

- Test file name: `<what>.<condition>.test.ts` or `<what>.spec.ts`.
- `describe` / `it` blocks. Name behaviour: `it('returns null when input is empty')`.
- Typed mocks: `vi.fn<Args, Return>()` (vitest) or `jest.fn()` typed via generics.
- `expect.assertions(N)` in async tests — prevents silent pass on missed await.
- Fixtures: typed constants in `__fixtures__/` or co-located `*.fixtures.ts`.

## Do / Don't

- DO: run `tsc --noEmit` before declaring green — type errors are bugs.
- DO: check `tsconfig.json` paths/aliases before adding imports — broken at emit.
- DON'T: use `any` to silence type errors — fix the type.
- DON'T: commit with `@ts-ignore` without explaining why.
- DON'T: mock modules not at system boundary.
