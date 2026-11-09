# Plan (normalize + plan, no browser)

Turn each assigned CSV row into one plan file. Do not open the browser or call any MCP tool. The live check happens in Generate. Do not guess.

## Input

- CSV at `TEST_CASES_CSV` (default `test-data/test-cases.csv`). Headers, case-insensitive and trimmed: `test case id`, `test scenario`, `expected result`, `roles`.
- Parse with a real CSV parser (quoted commas and newlines). Row number: header = row 1.
- Plan only your assigned TC-IDs. The main agent already skipped unchanged rows and flagged duplicate TC-IDs.

## Rules

1. TC-ID and scenario stay verbatim; they become the test title.
2. One step = one user action. Split "fill the form and submit".
3. Steps come from the scenario. Add a step only if the scenario cannot happen without it, and mark it `(implied)`.
4. Assertions come from the expected result, one per observable fact, each quoting its source clause. Every clause maps to at least one assertion. Never add one the expected result does not state.
5. Assertions are observable UI facts: visible text, element state, URL, count, value. Translate "successfully"/"correctly" only when the scenario defines it; otherwise flag.
6. Name UI elements in the CSV's words. Never invent accessible names or selectors.
7. Roles: split on `;` `,` `|` or newline. Slug = lowercase, non-alphanumerics to `-`. Do not invent, merge, or rename roles.
8. No credentials, emails, or passwords in plans.
9. No extra scenarios. Edge-case ideas go under `## Suggested extra coverage`, not into steps.

## Flag instead of planning

Return the row as "Needs clarification" (do not write the file yourself) when:

- TC-ID missing; scenario, expected result, or roles empty.
- Expected result not observable ("works as expected", "no issues").
- Scenario needs data, values, or state the row does not give.
- Steps and expected result contradict, or depend on another row's outcome.
- Needs non-browser evidence (email, database, PDF content, timing) with no UI evidence named.
- Role unclear (`Maker/Checker`: one role or two?).

Columns: `| TC-ID | Row | Problem | Question for the owner | Proposed reading (NOT applied) |`

## Output: write `specs/<TC-ID>.md` directly

```markdown
<!-- status: draft -->
<!-- roles: <role-slug>, ... -->
<!-- source: <csv path>, row <n>, hash <sha256(id|scenario|expected|roles)> -->
# <TC-ID> <scenario>

- Suite: <csv-stem>
- Seed: tests/seed.spec.ts
- File: tests/<csv-stem>/<tc-id-slug>.spec.ts
- Roles: <role name> (`<role-slug>`), ...
- Expected result (verbatim): "<cell>"
- Preconditions: signed in as <role>, fresh app state. <other stated state>

## Steps
1. <action>
   - A1. <check> (from: "<clause>")
2. <action>

## Suggested extra coverage
- <idea>
```

Each `A<n>` sits under the step after which it is checked. Return the plans written and any flagged rows in the return format given in your prompt.
