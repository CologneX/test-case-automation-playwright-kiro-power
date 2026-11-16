# Results

`tests/support/results-reporter.ts` (plain code, no AI) runs on every `playwright test` and writes one folder per run. Error sentences come from `tests/support/error-text.ts` (rule-based). The optional post-run `describe` step (`describe.md`) adds AI descriptions to the same folder:

```
reports/
  index.html            every run, newest first
  .last-run.json        used by --last-failed
  <YYYY-MM-DD_HH-mm-ss>/  (-2 suffix if two runs start in the same second)
    results.csv         one row per CSV test case and role, UTF-8 with BOM (Excel)
    results.json        run, totals, authFailures, results[]; schemaVersion 1
    summary.html        one page; rows link into playwright-report/
    playwright-report/  test-results/
```

| Status | Meaning |
|---|---|
| `passed` / `flaky` | Passed (flaky = passed on retry; investigate). |
| `failed` / `timed out` / `interrupted` | Did not pass. |
| `fixme` | Healer marked the app broken. See `test-data/heal-report.md`. |
| `blocked` | Role sign-in setup failed. See `authFailures`. |
| `skipped` | Skipped by the test itself. |
| `not run` | Spec exists but was filtered out (`--grep`, `--project`). |
| `not automated` | No spec yet. |
| `needs clarification` | Listed in `test-data/needs-clarification.md`. |

- CSV columns: `test case id`, `test scenario`, `expected result`, `role`, `status`, `duration (s)`, `retries`, `failed at`, `error`, `spec file`, `trace`, `screenshot`, `video`, `technical error`, `description by`.
- `trace`, `screenshot` and `video` are recorded for every test (`use: { trace: 'on', screenshot: 'on', video: 'on' }`; sign-in setup projects record nothing).
- `error` = the text to read. Without a describe run it is the rule-based sentence from `error-text.ts` (a message written in the test, `expect(locator, 'message')`, is kept as written). After `bun run describe` it is the AI description for described rows.
- `technical error` = the original Playwright error (first lines of each error, redacted).
- `description by` = `rules` (sentence from `error-text.ts`), or the model id (for example `google.gemma-4-e2b`) for an AI description. Empty for rows with no error.
- `results.json` per row: `error`, `technicalError`, and after describe also `aiDescription`, `aiModel`.
- `summary.html`, failed rows: the "Failed at / error" cell shows `failed at`, then the AI description (marked "AI description", model name, "may be inaccurate") or, if none, the rule-based sentence. A collapsed "technical error" holds the rule-based sentence (when the AI text is shown) and the Playwright error.
- AI descriptions are optional, made after the run, and never change a status. They can be wrong: for a decision, check the technical error and the Playwright report.
- "Failed at" = nearest `// A<n>.` or `// <n>.` comment above the failing line.
- Pass rate = (passed + flaky) / (passed + flaky + failed + timed out + interrupted).
- Role passwords are replaced with `***`. Only the `BASE_URL` origin is recorded.
- Runs are kept forever unless `REPORTS_KEEP=<n>` is set. `reports/` is gitignored.
- Open a report: `bun run report` (newest) or `bunx playwright show-report reports/<run>/playwright-report`. Traces need this served mode, not `file://`.

## When the user asks for results

Read `results.json` in the newest `reports/<run>/` folder (names sort by time), unless the user names a run. To compare runs, read each `results.json`. Do not re-run tests or parse console output. Report:

1. Run status, totals, pass rate.
2. Sign-in setup failures.
3. Each `failed`, `timed out`, `blocked`, `fixme` row: TC-ID, role, failed at, error (`aiDescription` if present, else `error`), and `technicalError` when the user asks for detail. Say when the text is an AI description.
4. `flaky` rows.
5. Paths to `reports/<run>/summary.html` and `reports/index.html`.
6. If the run has failed rows without `aiDescription` and the user has not been asked about AI descriptions in this project, offer them once (`describe.md`). Do not run `describe` without the user's choice of AWS profile.
