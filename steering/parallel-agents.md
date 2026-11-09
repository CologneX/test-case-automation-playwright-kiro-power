# Parallel sub-agents (main agent only)

The main agent splits, starts, merges, and reports. It does no item work while sub-agents run.

- Max 5 sub-agents at once, never more than work items. The user may set a lower number. Queue the rest; start the next when one finishes.
- Only the main agent writes `test-data/needs-clarification.md`, `test-data/heal-report.md`, `.auth/`, and `reports/`, once per step after all sub-agents finish.
- Sub-agents never ask the user. The main agent asks their questions, grouped by TC-ID, then starts one sub-agent per answered item.

## Sub-agent prompt (keep it this short)

```
Step: <Plan|Generate|Heal>. Read only: <files below> in <absolute Power steering path>.
Project: <absolute project root>. CSV: <path>.
Items: <TC-IDs | plan files | failing spec file + role project>.
Return exactly:
## Done
- <TC-ID>: <file>, <result>
## Needs clarification
| TC-ID | Row | Problem | Question for the owner | Proposed reading (NOT applied) |
## Heal report
| TC-ID | Date | Failing step / assertion | Expected (CSV) | Actual | Evidence |
## Questions for the user
- <TC-ID>: <question>
## Browser conflict
- none | <TC-ID>: <what was seen>
Keep empty sections' headings.
```

Files: Plan = `agent-planner.md`. Generate = `agent-generator.md`, `test-conventions.md`. Heal = `agent-healer.md`, `test-conventions.md`. Omit the "Heal report" section for Plan and Generate.

## Plan

1. Read the CSV once. Skip rows whose plan's source hash is unchanged. Flag missing and duplicate TC-IDs yourself.
2. Changed row: re-plan it; the new plan is `status: draft`. Row deleted from the CSV: list its plan and spec for the user; delete nothing without asking.
3. Split the rest into up to 5 contiguous, equal batches.
4. Rewrite `needs-clarification.md` with the merged rows (drop rows that are now clear), header:
   `| TC-ID | Row | Problem | Question for the owner | Proposed reading (NOT applied) |`
5. Report rows read, planned, flagged. Ask the user to review plans and change `status: draft` to `status: approved`. Feasibility is checked in Generate.

## Generate and Heal

1. Sign in once for every role in the items: `bunx playwright test --project=setup-<slug> ...`. All must pass; if one fails, follow `microsoft-auth.md` and start nothing for that role. While the session is fresh, setup projects run by MCP tools reuse it and write nothing.
2. Sub-agent CLI runs use `--no-deps --reporter=list`, so they never sign in or write `reports/`.
3. If the step outlasts `AUTH_MAX_AGE_MINUTES`: stop starting, let running ones finish, repeat step 1, continue.
4. Generate: one sub-agent per approved plan, queue ordered by role. Heal: one per failing test. Generate failures go to Heal.
5. Merge "Needs clarification" into `needs-clarification.md` and "Heal report" into `heal-report.md` (header: `| TC-ID | Date | Failing step / assertion | Expected (CSV) | Actual | Evidence (trace/screenshot path) |`).
6. Report per TC-ID: written, healed, `fixme`, needs clarification, or conflict re-run.

## Browser conflict (unverified: parallel sub-agents may share one page)

A sub-agent stops its item and reports a conflict when it sees a page or URL it did not open, log steps it did not perform, or a "page/target closed" error it did not cause. On any conflict: start no new sub-agents, let running ones finish, re-run conflicted items one at a time, and tell the user Generate and Heal are now sequential. Plan stays parallel.
