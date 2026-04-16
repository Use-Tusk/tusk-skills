---
name: tusk-drift-analyze
description: Analyze Tusk Drift API test deviations locally — classifies deviations as intended, unintended, or unrelated, and optionally fixes regressions.
allowed-tools: Bash(tusk drift run:*), Bash(git diff:*), Bash(git rev-parse:*)
---

You are performing local deviation analysis for Tusk Drift. Your job is to analyze API test deviations, classify them, and optionally fix regressions.

---

## Phase 1: Run Tests and Collect Deviations

Try running against Tusk Drift Cloud first:

```bash
tusk drift run --cloud --save-results agent --print
```

**If the command fails with an authentication error** (e.g., `Error: not authenticated. Run tusk auth login or set TUSK_API_KEY`):

1. Tell the user they need to authenticate and give them two options:
   - Run `tusk auth login` to authenticate interactively
   - Set the `TUSK_API_KEY` environment variable
2. Check if local traces exist at `.tusk/traces/`. If trace files are present, ask the user:
   > You're not authenticated with Tusk Drift Cloud, but I found local traces in `.tusk/traces/`. Would you like to replay those instead?
3. If the user wants to replay local traces, drop the `--cloud` flag for **all** subsequent `tusk drift run` commands in this session:

   ```bash
   tusk drift run --save-results agent --print
   ```

Capture the output directory path printed to stderr (e.g., `.tusk/results/run-20260329-140532/`).

---

## Phase 2: Read and Understand Deviations

1. **Read `index.md`** in the output directory to get the full picture: how many tests ran, how many passed, how many failed, and the list of deviation files.

2. **If there are no deviations**, report that all tests passed and stop.

3. **Determine the base branch:**

   - Check `index.md` for a `Base Branch:` line. If present, use that value.
   - If not present, fall back: try `main`, then `master` (check which exists with `git rev-parse --verify`).

4. **Get the PR diff** to understand what this branch changed:

   ```bash
   git diff <base-branch>...HEAD --stat
   git diff <base-branch>...HEAD
   ```

   If the diff is very large, use `--stat` first, then selectively read diffs for relevant files.

---

## Phase 2.5: Triage (10+ Deviations)

If there are 10 or more deviations, do NOT jump straight into analyzing each one individually. Instead, triage first:

1. **Scan frontmatter only** across all deviation files to build an overview. Read just the YAML frontmatter (between the `---` markers) from each file — do not read the full body yet.

2. **Group by endpoint** — deviations on the same endpoint likely share a root cause. You only need to deeply analyze one representative per group, then apply the finding to the rest.

3. **Fast-classify obvious cases** — if `has_mock_not_found: true` and the endpoint does not appear in the PR diff, classify as UNRELATED without deep investigation. These are mock gaps, not code bugs.

4. **Prioritize** the remaining deviations:
   - RESPONSE_MISMATCH on endpoints touched by the PR diff — most likely to be real intended/unintended changes. Analyze these first.
   - NO_RESPONSE failures — investigate for crashes/deadlocks.
   - Everything else — handle after the high-priority ones.

Only read the full deviation file body for deviations that require deep investigation.

---

## Phase 3: Analyze Each Deviation

For each deviation (or group of similar deviations) that requires analysis:

### 3a. Read the deviation file

Read the full `.md` file. Pay attention to:

- The **frontmatter** (endpoint, failure_type, status codes, has_mock_not_found)
- The **Response Diff** section (what changed in the API response)
- The **Outbound Call Context** table (mock match quality)
- The **Mock Not Found Events** section (missing mocks)

### 3b. Investigate the codebase

- Find the server endpoint implementation for the endpoint in the deviation
- Cross-reference with the PR diff to see what changed
- Trace the execution path that would lead to the observed deviations
- Check if outbound call changes (new calls, removed calls) explain the deviation

### 3c. Classify the deviation

Assign one of these types:

| Type           | Meaning                                                                                                                                                                     |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **INTENDED**   | The deviation directly aligns with what the PR is changing. Example: PR adds a `verification_required` field, and the deviation shows that field appearing in the response. |
| **UNINTENDED** | The deviation looks like a regression not matching the PR's intent. Example: PR refactors auth middleware, and an unrelated endpoint starts returning 500.                  |
| **UNRELATED**  | The deviation is caused by environmental factors, mock issues, or flaky behavior unrelated to PR changes.                                                                   |
| **UNKNOWN**    | Cannot determine with available information.                                                                                                                                |

### 3d. Apply these heuristics

**Mock-related heuristics (critical):**

- If `has_mock_not_found: true` and the PR did NOT intentionally introduce the new outbound call, the deviation is likely **UNRELATED** — the missing mock is causing the deviation, not a code bug.
- If outbound calls matched at FALLBACK level, the response may differ due to imprecise mocking rather than a real regression. Note this and factor it into your confidence.
- If outbound calls matched at FUZZY or GLOBAL scope (from other traces), the mock data may not be representative. Lower your confidence.

**NO_RESPONSE heuristics:**

- During replay, ALL external dependencies (databases, HTTP calls, Redis, gRPC) are mocked and return instantly.
- Database query performance CANNOT cause timeouts during replay.
- Network latency CANNOT cause timeouts during replay.
- If `failure_type: NO_RESPONSE`, look for infinite loops, crashes, deadlocks, or panics in the application code itself.

**General investigation guidance:**

- Start by examining the server endpoint implementation
- Check the PR diff to understand what changed
- Check stack traces in outbound call context to understand call flow
- Consider how poor mock matches might cascade to server-level deviations
- Follow the execution path leading to the observed deviations

### 3e. Write your analysis

For each deviation, produce:

- **Classification**: INTENDED / UNINTENDED / UNRELATED / UNKNOWN
- **Confidence**: HIGH / MEDIUM / LOW
- **What changed**: 1-2 sentences describing the deviation
- **Why it changed**: 2-4 sentences explaining the root cause, citing specific file paths and line numbers
- **Potential fix** (if UNINTENDED): 1 sentence describing what to fix and where

---

## Phase 4: Present Results

After analyzing all deviations, present a summary table:

```markdown
## Deviation Analysis Summary

| #   | Endpoint             | Classification | Confidence | Root Cause                                                            |
| --- | -------------------- | -------------- | ---------- | --------------------------------------------------------------------- |
| 1   | POST /api/v1/users   | INTENDED       | HIGH       | PR adds verification_required field to user creation response         |
| 2   | GET /api/v1/orders   | UNRELATED      | HIGH       | Mock not found for shipping-service; missing mock, not a code bug     |
| 3   | PUT /api/v1/settings | UNINTENDED     | MEDIUM     | Auth middleware refactor broke header propagation in settings handler |
```

Then provide detailed analysis for each deviation.

If there are any **UNINTENDED** deviations, ask the user:

> Found N unintended deviation(s) that may be regressions. Would you like me to attempt to fix them?

Wait for the user's response before proceeding to Phase 5.

---

## Phase 5: Fix Unintended Deviations (User-Initiated)

Only proceed if the user confirms they want fixes.

For each UNINTENDED deviation:

### 5a. Investigate and fix

- Look at the relevant code locations identified in Phase 3
- Propose and apply a code fix
- Explain what you changed and why

### 5b. Verify the fix

Re-run the specific test (drop `--cloud` if replaying local traces):

```bash
tusk drift run --cloud --save-results agent --print --trace-id {deviation_id}
```

### 5c. Evaluate the result

After re-running, read the new output and determine the outcome:

| Outcome                 | Meaning                             | Action                                                                                                               |
| ----------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **FIXED**               | Test passes now                     | Report success, move to next deviation                                                                               |
| **DEVIATION_IDENTICAL** | Same deviation persists after fix   | The code change had no effect. Likely a mock gap or environmental issue. Stop and reclassify as UNRELATED/UNFIXABLE. |
| **DEVIATION_CHANGED**   | Different deviation now             | Assess whether new deviation is a separate issue. Report it, don't chase it.                                         |
| **NEW_DEVIATION**       | Original fixed but new one appeared | Flag the new deviation as separate. Don't chase it in this loop.                                                     |

### 5d. Iterate if needed

- Max **3 fix attempts** per deviation before giving up
- **Early stop**: if the same deviation persists unchanged after 2 different fix approaches, stop early — the issue is likely a mock gap or environment factor, not a code bug. Don't use the third attempt.
- If a fix resolves one deviation but introduces a new one, flag it and stop

### 5e. Report fix results

Present a summary of fix outcomes:

```markdown
## Fix Results

| #   | Endpoint             | Outcome              | Details                                                            |
| --- | -------------------- | -------------------- | ------------------------------------------------------------------ |
| 1   | PUT /api/v1/settings | FIXED                | Fixed header propagation in auth middleware                        |
| 2   | DELETE /api/v1/cache | UNFIXABLE_MOCK_ISSUE | Deviation caused by missing mock for cache-service, not a code bug |
```

Outcome values:

- **FIXED**: Code change resolved the deviation, test passes
- **PARTIALLY_FIXED**: Some deviations resolved but others remain
- **UNFIXABLE_MOCK_ISSUE**: Deviation caused by missing or imprecise mock, not a code bug
- **UNFIXABLE_INTENDED**: On closer inspection, deviation is actually intended behavior
- **UNFIXABLE_FLAKY**: Deviation is intermittent or caused by replay environment factors
- **GAVE_UP**: Exhausted attempts without clear diagnosis

Include reasoning for each outcome.
