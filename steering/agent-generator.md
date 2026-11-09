# Generate (approved plan → spec)

Turn each assigned plan into one spec by replaying it live. Apply `test-conventions.md`. Tools: `generator_setup_page`, `generator_read_log`, `generator_write_test`, `browser_*` (actions), `browser_verify_*` (assertions).

1. Only plans whose first line is `<!-- status: approved -->`. Skip and list drafts.
2. `generator_setup_page` with `plan` = full plan text, `seedFile: "tests/seed.spec.ts"`, `project: "<first role slug>"`. The page starts signed in.
3. Execute every step live, using the step text as the intent of each tool call. Check every `A<n>` with a `browser_verify_*` tool so the log records the `expect`.
4. A step or assertion cannot be performed or observed as written: stop this plan. Do not change the step or expected value. Return it as "Needs clarification" with what you saw.
5. `generator_read_log`, then immediately `generator_write_test` with `fileName` = the plan's `File`. Before writing, apply `test-conventions.md` to the logged code.
6. Run `bunx playwright test <file> --project=<role-slug> --no-deps --reporter=list` for each row role. Failure: report it as failed (the main agent starts Heal).
7. Do not write `test-data/`, `.auth/`, or `reports/`. Do not ask the user. Use the return format given in your prompt.

Browser conflict: if you see a page or URL you did not open, log steps you did not perform, or a "page/target closed" error you did not cause, stop this item and report it under "Browser conflict".
