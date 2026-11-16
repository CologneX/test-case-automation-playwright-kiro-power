# Describe (optional, after a run)
`tests/support/describe-failures.ts` asks Gemma 4 E2B on Amazon Bedrock to describe each failed row in 2 or 3 plain sentences. It runs after a test run, by hand (`bun run describe`). Never run it inside `playwright test`, never import it from a spec or the config. The Power rule stays: AI authors and heals tests, tests never call an LLM.
**The agent must never default to a profile. Never use `default`, or any profile the user did not choose.**
## What it does
- Reads `results.json` of a run folder. For each `failed`, `timed out`, or `interrupted` row it sends text only: test case id, scenario, expected result, role, failing step, the report's error sentence, and the `Clarification applied` notes of `specs/<TC-ID>.md`.
- Does not send screenshots, traces, page content, `.env`, or credentials.
- Writes the answer into the same run folder: `results.json` (`aiDescription`, `aiModel`), `results.csv` (`error` column = AI text, `description by` = model name), `summary.html` (AI text visible, the rest under a collapsed "technical error"). Pass/fail never changes.
- Fails soft. With no profile or expired credentials the script stops with exit code 1 before changing any file. A row the model cannot describe keeps its rule-based sentence. The run folder is always complete without descriptions.
- Endpoint: `https://bedrock-mantle.<region>.api.aws/openai/v1/chat/completions`, model `google.gemma-4-e2b`. Auth is a 15-minute bearer token made from the chosen profile with `@aws/bedrock-token-generator`, valid only in the region it was made for. Source: https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-google-gemma-4-e2b.html
- Supported regions: `us-east-1` (default), `us-east-2`, `us-west-2`, `eu-central-1` (also `us-gov-west-1`). Not `ap-southeast-1`.
- IAM actions the profile needs: `bedrock-mantle:CreateInference` and `bedrock-mantle:CallWithBearerToken` (managed policy `AmazonBedrockMantleInferenceAccess`).
## Agent procedure
Do this once per project, the first time the user has failed rows, or when onboarding points here. Do not ask again if `DESCRIBE_AWS_PROFILE` is already in `.env` or the user said no.
1. **Ask whether they want AI descriptions.** Say:
   - What is sent: the text listed above, about each failed row.
   - It leaves the machine for Amazon Bedrock, in a region that is probably not their usual one.
   - Screenshots and traces are not sent.
   - The descriptions are AI-generated and may be inaccurate. The technical error stays in the report.
   - Cost is per token and small. Only failed rows are sent.
   If no, stop. Tests and reports work without it.
2. **Ask which AWS profile to use.** List profile names only: run `aws configure list-profiles`. Never open `~/.aws/credentials` or `~/.aws/config`, and never print keys or tokens. Always offer `none / skip`. Use exactly this wording, with the real names:
   > Which AWS profile should I use for the AI descriptions? Profiles on this machine: `<name-1>`, `<name-2>`, ... Or reply "none" to skip. The profile needs permission for Amazon Bedrock inference (`bedrock-mantle:CreateInference` and `bedrock-mantle:CallWithBearerToken`). I will not pick one for you.
   If the list is empty or the command fails, say so and ask the user to name a profile or skip. Do not choose one.
3. **Ask which region.** Show the supported list, default `us-east-1`. Say that the failed-row text is processed in that region, so choose one that fits the project's data residency (for example `eu-central-1` for the EU).
4. **Write the choice.** Put `DESCRIBE_AWS_PROFILE=<name>` and `DESCRIBE_REGION=<region>` into the project's `.env` (names only, no secrets; change only those two lines). Make sure `.env.example` has `DESCRIBE_AWS_PROFILE=` and `DESCRIBE_REGION=` with empty values. Do not read other lines of `.env`.
5. **Check access.**
   - `bun run describe -- --dry-run` prints what would be sent and calls nothing. Show the user one printed row.
   - Then one real run: `bun run describe`. It prints `Sending the text of the failed rows (no screenshots) to Amazon Bedrock in <region> with AWS profile <name>.`
6. **Expired or missing credentials.** The script prints `Could not get AWS credentials for profile "<name>": <reason>`. Tell the user to sign in themselves: `aws sso login --profile <name>` (IAM Identity Center) or `aws login --profile <name>`. Then re-run. Never ask for access keys, secrets, or tokens.
7. **A different profile later.** Ask again with the wording in step 2. Pass it with `--profile <name>` for one run, or update `DESCRIBE_AWS_PROFILE` if the user wants it kept.
## Running it
The script runs with bun (`"describe": "bun tests/support/describe-failures.ts"`). If `bun -v` fails (the project uses npm only), tell the user to install bun before using this step. Tests and reports do not need it.
```bash
bun run describe                                  # newest run in reports/
bun run describe -- reports/<run>                 # a given run
bun run describe -- --profile <name>              # overrides DESCRIBE_AWS_PROFILE
bun run describe -- --region us-west-2            # overrides DESCRIBE_REGION (default us-east-1)
bun run describe -- --force                       # also rows that already have an AI description
bun run describe -- --dry-run                     # print what would be sent; no profile needed
```
- Without a profile the script prints `No AWS profile set. Nothing was sent.` and exits with code 1. This is correct. Ask the user (procedure step 2). Never add a profile to make it run.
- Rows that already have an AI description are skipped unless `--force`.
- A run folder made before `error-cell.ts` existed has no error-cell markers in `summary.html`. `results.json` and `results.csv` are still updated. Rerun the tests to get the new layout.
- The two sample sentences inside the `SYSTEM` prompt of the script show the model the style wanted. The user may replace them with sentences from their own test cases.
## Meaning in the report without AI
Two things shape the sentence in the `error` column before any AI is involved:
1. **Rule-based sentence.** `tests/support/error-text.ts` rewrites Playwright errors into one plain sentence, for example `Button matching /^Edit/ not found – could not click it (waited 10 s)`. It states only what the page showed or lacked. It does not know what it means for the product. Unknown error shapes fall back to the first lines of the Playwright text.
2. **A message in the test.** For a known product finding, write the meaning into the assertion: `await expect(locator, 'Redemption Amount (MYR) should be editable').toBeEditable();`. The reporter keeps that message as written. It is the first line of the error, so the rule-based sentence and the AI description both start from it. If the page showed another value, `(expected X, got Y)` is added.
A message sits on an assertion line, so the Heal rules apply: add or change one only when the user states the finding, and ask first.
