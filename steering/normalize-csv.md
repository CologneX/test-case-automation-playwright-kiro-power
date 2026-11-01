# Normalize CSV rows

Turn each CSV row into numbered steps and numbered assertions. Do not open the browser in this stage. Do not guess.

## Input

- CSV at `TEST_CASES_CSV` (default `test-data/test-cases.csv`).
- Match headers case-insensitively and trim them: `test case id`, `test scenario`, `expected result`, `roles`.
- Parse with a real CSV parser (quoted commas and newlines are common). Keep the source row number (header = row 1).

## Output per row: `test-data/normalized/<TC-ID>.md`

```markdown
# <TC-ID> <test scenario, verbatim>

- Source: <csv path>, row <n>
- Roles: <role name> (`<role-slug>`), ...
- Expected result (verbatim): "<expected result cell>"

## Preconditions
- Signed in as <role>. <other state the row states or clearly implies>

## Steps
1. <one user action, imperative, names the UI element in the user's words>
2. ...

## Assertions
- A1. <one observable check> (from: "<exact clause of the expected result>")
- A2. ...
```

## Rules

1. Keep the TC-ID and scenario text verbatim. They become the test title.
2. One step = one user action. Split "fill the form and submit" into separate steps.
3. Derive every step from the scenario text. Add a step only if the scenario cannot happen without it (for example, opening the page). Mark such steps `(implied)`.
4. Derive every assertion from the expected result. Split it into one assertion per observable fact. Each assertion quotes the clause it came from.
5. Every clause of the expected result maps to at least one assertion. Never add an assertion the expected result does not state.
6. Write assertions as observable UI facts: visible text, element state, URL, count, value. Translate "successfully", "correctly", "properly" only when the scenario defines what that looks like. Otherwise flag the row.
7. Split the roles cell on `;`, `,`, `|`, or newline. Slug = lowercase, non-alphanumerics replaced with `-`. Do not invent, merge, or rename roles.
8. Never write credentials, emails, or passwords into normalized files.

## Flag instead of guessing

Do not create a normalized file for a row that has any of these. Add it to `test-data/needs-clarification.md` instead:

- Missing or duplicate test case id.
- Empty scenario, expected result, or roles.
- Expected result is not observable ("works as expected", "system behaves correctly", "no issues").
- Scenario needs data, values, or state the row does not give (which record, which amount, which file).
- Steps or expected result contradict each other or depend on another row's outcome.
- Needs something the browser cannot check (email received, database value, PDF content, timing) and no UI evidence is named.
- Role is unclear (for example `Maker/Checker`: one role or two?).

## `test-data/needs-clarification.md` format

```markdown
# Needs clarification

| TC-ID | Row | Problem | Question for the owner | Proposed reading (NOT applied) |
|---|---|---|---|---|
```

Rewrite this file on every run. Remove rows that are now clear.

## Re-runs

- Store `Source hash: sha256(<id>|<scenario>|<expected>|<roles>)` under `Source`. On re-run, skip rows whose hash is unchanged.
- When a row changes, update its normalized file and set its plan in `specs/` back to `status: draft`.
- When a row is deleted from the CSV, list its spec and plan files for the user. Do not delete them without asking.

Finish by reporting: rows read, rows normalized, rows flagged.
