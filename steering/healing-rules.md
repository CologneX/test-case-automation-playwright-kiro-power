# Healing rules

These rules override `agent-healer.md`. Read them before healing any test.

## Allowed without asking

- Replace a locator with one discovered on the live app (`browser_snapshot`, `browser_generate_locator`) that targets the element with the same intent as the step.
- Fix waits: replace a fixed wait with a web-first assertion or auto-waiting action, add `await expect(locator).toBeVisible()` before an action, add `page.waitForURL(...)` after a navigation the step causes.
- Add `exact: true`, `.filter()`, or a parent scope to remove an ambiguous match.

## Forbidden without asking the user first

- Changing, loosening, removing, or adding an assertion (`expect` call, matcher, `.not`, expected value, regex).
- Changing a step's action, order, input data, or the test title or tags.
- Adding retries, `test.slow()`, longer global timeouts, `waitForTimeout`, `networkidle`, or `force: true` to make a test pass.
- Editing `specs/`, `test-data/normalized/`, or the CSV.

To ask: show the failing assertion, the actual value or state, the CSV expected result, and your proposed change. Wait for an answer.

## Decide: test broken or app broken

1. Element exists with the same purpose but a new name, role, or position: test broken. Fix the locator.
2. Step fails on timing and passes once synchronized: test broken. Fix the wait.
3. Assertion fails with a different value, text, or state, and the step itself ran: possibly app broken. Ask the user.
4. Element or feature the scenario needs is missing, errors, or behaves against the expected result: app broken. Do not fix it.

## When the app is broken

1. Mark the test `test.fixme()`. Do not touch its assertions.
2. Before the failing step, add `// FIXME(<date>): expected <X>, app shows <Y>.`
3. Add a row to `test-data/heal-report.md`:

```markdown
| TC-ID | Date | Failing step / assertion | Expected (CSV) | Actual | Evidence (trace/screenshot path) |
|---|---|---|---|---|---|
```

4. Report it to the user. Do not mark it fixed.

## Limits and verification

- Max 3 heal attempts per test. Then stop and report what you tried.
- After a fix, run the test with `--repeat-each=3`. It must pass all 3 times.
- Report every healed test: TC-ID, old locator or wait, new one, and why.
