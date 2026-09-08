# TypeScript / JavaScript — vitest, jest

## Detect and run

- `package.json` → check `devDependencies`: `vitest` or `jest` (also
  `vitest.config.ts` / `jest.config.*`). Run through the project's script
  (`npm test`, `pnpm test`) so config, path aliases, and setup files apply.
- One file: `npx vitest run path/to/file.test.ts` / `npx jest path/to/file`.
- One test: `-t "name substring"`. Watch mode is for humans, not for agents —
  use `vitest run` so the process exits.
- Coverage: `vitest run --coverage` / `jest --coverage`; read uncovered lines,
  not the total.

## Layout

Next to the source (`refund.ts` → `refund.test.ts`) or under `__tests__/` —
follow the repo. Keep `describe` for the unit and the test name for the
behaviour: `describe("refund", () => it("rejects an already settled order", …))`.

## Idioms

```ts
import { describe, expect, it, vi, beforeEach } from "vitest"; // jest: globals
import { refund } from "./refund";

describe("refund", () => {
  it("rejects an already settled order", () => {
    const order = makeOrder({ status: "settled" });
    expect(() => refund(order, 100)).toThrow(AlreadySettled);
  });

  it.each([
    { amount: 0, expected: 0, name: "zero" },
    { amount: 100, expected: 100, name: "at cap" },
  ])("passes $name through", ({ amount, expected }) => {
    expect(refund(makeOrder(), amount).amount).toBe(expected);
  });
});
```

- **Module mocks**: `vi.mock("./client")` / `jest.mock("./client")` are hoisted
  above imports — a factory referencing an outer variable throws. Prefer
  dependency injection; reach for module mocks when the import is not injectable.
- **Reset between tests**: `beforeEach(() => vi.clearAllMocks())`, or
  `restoreMocks: true` in config. Leaked mock state is the top cause of
  order-dependent failures here.
- **Time**: `vi.useFakeTimers()` + `vi.setSystemTime(new Date("2026-01-01"))`,
  and `vi.useRealTimers()` in cleanup. `await vi.advanceTimersByTimeAsync(1000)`
  instead of sleeping.
- **Async**: always `await` the assertion — `await expect(p).rejects.toThrow(...)`.
  A forgotten `await` produces a test that passes while asserting nothing.
- **Objects**: `toEqual` for structure, `toBe` for identity/primitives,
  `toMatchObject` when only some fields matter. Avoid snapshots for logic — they
  record whatever the code does today, including the bug.
- **Types**: import the real types; a test that only compiles because of `any`
  will not catch a signature change.

## Traps

- `expect(fn).toThrow()` needs a function, not a call: `expect(() => fn())`.
- `toBe` on objects compares references and fails confusingly.
- `jest.mock` with a partial factory silently drops the other exports —
  `jest.requireActual` / `importActual` to spread the real module first.
