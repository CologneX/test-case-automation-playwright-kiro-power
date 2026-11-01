# Test conventions

These rules override the ported agent files.

## Files and titles

- One CSV row = one test = one file: `tests/<csv-stem>/<tc-id-slug>.spec.ts`.
- `test.describe('<csv-stem>', ...)` wraps the test. `<csv-stem>` is the CSV file name without extension.
- Test title is exactly `<TC-ID> <test scenario>`, both verbatim from the CSV.
- Tag the test with every role from the row: `{ tag: ['@role-<role-slug>', ...] }`. The role projects select tests by this tag.
- Import from the fixtures file: `import { test, expect } from '../support/fixtures';`
- Start each file with:

```ts
// spec: specs/<TC-ID>.md
// seed: tests/seed.spec.ts
// csv: <csv path>, row <n>
```

## Steps and assertions

- Before each step's code, write `// <n>. <step text>` from the plan. Do not repeat it for multi-action steps.
- Before each assertion, write `// A<n>. <assertion text>`. The results reporter uses these comments to fill the "failed at" column. Never remove them.
- Assertions map 1:1 to the plan's assertions, which map to the CSV expected result. No extra assertions. No missing ones.
- Use web-first assertions only: `await expect(locator).toBeVisible()`, `toHaveText`, `toHaveValue`, `toHaveURL`, `toHaveCount`, etc. Never assert on `await locator.textContent()` or `isVisible()` results.
- Expected values come verbatim from the plan. Use a regex only for data that is dynamic by nature (dates, generated ids), and say so in a comment.

## Locators

Use the first one that works, discovered on the live app (`browser_snapshot`, `browser_generate_locator`):

1. `getByRole(role, { name })`
2. `getByLabel`
3. `getByPlaceholder`
4. `getByText`
5. `getByTestId`
6. CSS or XPath: last resort only, with a `// why:` comment.

- Scope with `.filter()` or a parent locator instead of `.nth()`. Use `.first()` or `.nth()` only when order is part of the scenario.
- Use `exact: true` when a name is a prefix of another name.

## Speed and determinism

- Never use `page.waitForTimeout`, `waitForLoadState('networkidle')`, `page.$`, `page.$$`, `force: true`, or `test.only`. Rely on auto-waiting.
- Never sign in inside a test. The role project's storageState does it.
- Tests are independent and parallel-safe. No shared state, no order dependencies, no `test.describe.serial`.
- Data a test creates is prefixed with its TC-ID and `test.info().parallelIndex`, and cleaned up in the same test when the app allows it.
- No LLM, AI SDK, or MCP calls in test code. No `console.log` left behind.
