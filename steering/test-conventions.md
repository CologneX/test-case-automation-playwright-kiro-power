# Test conventions

## File

- One row = one test = `tests/<csv-stem>/<tc-id-slug>.spec.ts`, wrapped in `test.describe('<csv-stem>', ...)`.
- Title exactly `<TC-ID> <test scenario>`, verbatim from the CSV.
- Tags: every row role, `{ tag: ['@role-<role-slug>', ...] }`. Role projects select by tag.
- `import { test, expect } from '../support/fixtures';`
- First lines:

```ts
// spec: specs/<TC-ID>.md
// seed: tests/seed.spec.ts
// csv: <csv path>, row <n>
```

## Steps and assertions

- `// <n>. <step text>` before each step's code (once per step). `// A<n>. <assertion text>` before each assertion. The reporter reads these for "failed at"; never remove them.
- Assertions map 1:1 to the plan's `A<n>`. None extra, none missing.
- Web-first only: `await expect(locator).toBeVisible()`, `toHaveText`, `toHaveValue`, `toHaveURL`, `toHaveCount`. Never assert on `textContent()` or `isVisible()` results.
- Expected values verbatim from the plan. Regex only for dynamic data (dates, generated ids), with a comment.

## Locators (first that works, found live)

`getByRole(role, { name })` > `getByLabel` > `getByPlaceholder` > `getByText` > `getByTestId` > CSS/XPath with a `// why:` comment. Scope with `.filter()` or a parent instead of `.nth()` unless order is part of the scenario. `exact: true` when a name prefixes another.

## Never

`page.waitForTimeout`, `networkidle`, `page.$`, `page.$$`, `force: true`, `test.only`, `test.describe.serial`, sign-in inside a test, shared state between tests, LLM/AI SDK/MCP calls, leftover `console.log`.

Data a test creates is prefixed with its TC-ID and `test.info().parallelIndex`, and cleaned up in the same test when the app allows.
