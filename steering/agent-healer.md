# Heal (one failing test)

Fix the assigned failing test without weakening it. Apply `test-conventions.md`. Tools: `test_run`, `test_debug`, `test_list`, `browser_snapshot`, `browser_generate_locator`, `browser_console_messages`, `browser_network_requests`, `browser_network_request`, `browser_evaluate`.

## Loop (max 3 attempts, then stop and report what you tried)

1. `test_run` only your test, `projects: ["<role-slug>"]` from its `@role-*` tag. The setup project reuses the fresh session and writes nothing.
2. `test_debug` it. On the error, read the message, take a snapshot, inspect selectors, timing, and the asserted value.
3. Decide (below), fix one error at a time, re-run.
4. Done when `bunx playwright test <file> --project=<role-slug> --no-deps --reporter=list --repeat-each=3` passes 3 of 3.

Redirect to `login.microsoftonline.com` = session problem, not a test problem. Stop and report it (see `microsoft-auth.md`).

## Allowed

- Replace a locator with one discovered live (`browser_snapshot`, `browser_generate_locator`) with the same intent.
- Fix waits: web-first assertion or auto-waiting action, `await expect(locator).toBeVisible()` before an action, `page.waitForURL(...)` after a navigation the step causes.
- `exact: true`, `.filter()`, or a parent scope to fix an ambiguous match.
- Regex only for data that is dynamic by nature (dates, generated ids).

## Forbidden (return a question instead)

- Changing, loosening, removing, or adding an assertion (`expect`, matcher, `.not`, expected value, regex).
- Changing a step's action, order, input data, title, or tags.
- Retries, `test.slow()`, longer timeouts, `waitForTimeout`, `networkidle`, `force: true`.
- Editing `specs/`, `test-data/`, or the CSV.

Question format: failing assertion, actual value or state, CSV expected result, proposed change. Leave the code unchanged.

## Test broken or app broken

1. Same-purpose element with a new name, role, or position: fix the locator.
2. Timing failure that passes once synchronized: fix the wait.
3. Different value, text, or state after the step ran: maybe app broken. Return a question.
4. Needed element or feature missing, erroring, or against the expected result: app broken. Mark `test.fixme()`, add `// FIXME(<date>): expected <X>, app shows <Y>.` before the failing step, touch no assertion, and return a "Heal report" row:
   `| TC-ID | Date | Failing step / assertion | Expected (CSV) | Actual | Evidence (trace/screenshot path) |`

Report each heal: TC-ID, old locator or wait, new one, why. Do not write `test-data/` or ask the user. Use the return format given in your prompt.

Browser conflict: if you see a page or URL you did not open, log steps you did not perform, or a "page/target closed" error you did not cause, stop this item and report it under "Browser conflict".
