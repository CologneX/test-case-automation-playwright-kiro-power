# Results

`tests/support/results-reporter.ts` (plain code, no AI) runs on every `playwright test` and writes one folder per run:

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

- "Failed at" = nearest `// A<n>.` or `// <n>.` comment above the failing line.
- Pass rate = (passed + flaky) / (passed + flaky + failed + timed out + interrupted).
- Role passwords are replaced with `***`. Only the `BASE_URL` origin is recorded.
- Runs are kept forever unless `REPORTS_KEEP=<n>` is set. `reports/` is gitignored.
- Open a report: `bun run report` (newest) or `bunx playwright show-report reports/<run>/playwright-report`. Traces need this served mode, not `file://`.

## When the user asks for results

Read `results.json` in the newest `reports/<run>/` folder (names sort by time), unless the user names a run. To compare runs, read each `results.json`. Do not re-run tests or parse console output. Report:

1. Run status, totals, pass rate.
2. Sign-in setup failures.
3. Each `failed`, `timed out`, `blocked`, `fixme` row: TC-ID, role, failed at, error.
4. `flaky` rows.
5. Paths to `reports/<run>/summary.html` and `reports/index.html`.
