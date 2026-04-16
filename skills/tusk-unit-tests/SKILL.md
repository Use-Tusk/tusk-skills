---
name: tusk-unit-tests
description: Discover and selectively adopt existing Tusk-generated unit tests. Only relevant once a branch has been pushed and a PR exists — Tusk generates runs in CI on PR push, not for unpushed local work. Use when iterating on a PR, reviewing CI, or when the user asks about adding the Tusk-generated tests for their current branch.
when_to_use: |
  Trigger phrases: "review Tusk tests", "apply generated tests", "add Tusk tests",
  "CI failed on tests", "what tests did Tusk write for me".
  Also invoked from the iterate-pr skill after CI passes.
allowed-tools: Bash(tusk unit latest-run:*), Bash(tusk unit get-run:*), Bash(tusk unit get-scenario:*), Bash(tusk unit get-diffs:*), Bash(tusk unit retry:*), Bash(tusk unit feedback:*), Bash(git diff:*), Bash(git status:*), Bash(git rev-parse:*), Bash(git log:*), Bash(git apply:*), Bash(jq:*)
---

# Tusk Unit Tests

Discover, reason about, and selectively adopt existing Tusk-generated unit tests for the current branch. Tusk generates tests in CI on PR push — this skill surfaces them, checks whether each test is still applicable to the current code state, and helps the user apply the ones worth keeping.

**Scope**: this skill assumes a pushed branch with a PR. If the user is on a fresh local branch with no PR yet, the skill will call `latest-run`, get a "no PR exists" message, and stop — that's the correct behavior, not a bug. Tusk doesn't generate tests for unpushed work.

**Requires**: `tusk` CLI installed and authenticated (`tusk auth login`). If either is missing, the first command will fail — stop and tell the user how to fix it.

---

## Phase 1 — Applicability gate

1. If `git rev-parse --abbrev-ref HEAD` returns `main`/`master` (or the repo's default), exit silently.
2. Capture `git rev-parse HEAD` (local HEAD sha, used for Phase 3's freshness diff and Phase 4's retry check) and `git rev-parse @{u} 2>/dev/null` (upstream tracking sha, used for Phase 4's retry check — may fail if no upstream).

Do not skip the remaining phases on "ahead of origin" or dirty working tree — still fetch the run and let the user see the freshness gap.

---

## Phase 2 — Fetch the latest run

```bash
tusk unit latest-run
```

Infers repo and branch from the git remote. Returns `{ latest, history }` where `latest` is a run summary (no scenarios). **On HTTP 404**, the backend returns an actionable explanation (draft PR, unseated author, repo disabled, webhook pending, etc.) — relay it verbatim and stop. Do not reword.

On success, inspect `latest.status`:

| Status | Action |
|---|---|
| `in_progress` | Tusk is still generating. Tell the user to re-run shortly. Stop. |
| `error` / `cancelled` / `skipped` | Show `latest.status_detail`. Suggest `tusk unit retry --run-id <id>` only if Phase 4's retry condition holds. Stop. |
| `completed` | Fetch full scenarios with `tusk unit get-run <run_id>`, then continue to Phase 3. |

---

## Phase 3 — Per-scenario freshness

For each scenario in `get-run`'s `test_scenarios` array, run:

```bash
git diff <run.commit_sha>..HEAD -- <scenario.file_path>
```

Classify:

| Classification | Condition |
|---|---|
| **FRESH** | File untouched since the run sha. |
| **LIKELY FRESH** | File touched, but `scenario.symbol_name` does not appear in any diff hunk. Soft classification — grep also matches signature lines and call sites, so if in doubt, bump to NEEDS REVIEW. |
| **NEEDS REVIEW** | `scenario.symbol_name` appears in a diff hunk. Read the generated test and the diff together and judge whether the assertions still hold. |
| **STALE** | File deleted since the run sha. |

Separately, judge whether each test is valuable in context — do not apply fixed rules like "pure functions = high value."

---

## Phase 4 — Present and decide

Scale the presentation to the number of scenarios. Don't mechanically list every scenario with a per-row table — that's fine for ~10 scenarios, useless for 40.

- **All scenarios are FRESH and look valuable** → lead with a one-line verdict ("all N scenarios look good — want me to apply them?") and let the user say yes. No table needed unless they ask for details.
- **A handful look bad, rest look good** → summarize the good ones collectively ("N tests in X, Y, Z look valuable") and call out the specific exceptions with their reasoning. Ask the user to confirm the shape of the adoption.
- **Mixed or ambiguous** → a per-scenario table (or grouped by file) is appropriate. Columns: file, symbol, freshness, recommendation, one-sentence reason.
- **Large runs (20+ scenarios)** → always group by file or by freshness/value classification. A 40-row table nobody reads is worse than a 5-line summary with the outliers called out.

Match the shape to what the user needs to decide. The goal is an informed adopt/skip decision, not a completionist audit.

**Surface the freshness gap only when it matters.** If the run sha matches local HEAD and the working tree is clean, don't belabor it — skip straight to the scenarios. Only call out the gap when:

- The run was generated against a commit that is not the user's current HEAD (tests may reference code that has since moved).
- The user has unpushed commits or a dirty working tree (their WIP won't get a fresh run until they push).

Phrase it naturally in your response — no banners or fixed strings.

**Retry suggestion**: if ≥50% of scenarios are NEEDS REVIEW because the run targeted wrong symbols or used wrong mocks (not because the user made legitimate large changes), suggest a combined feedback + retry in one call — this tells Tusk *what* went wrong and asks for a fresh run in the same step:

```bash
tusk unit feedback --run-id <run-id> --retry --file - <<'EOF'
{
  "run_feedback": {
    "comment": "<short explanation of what went wrong — e.g., wrong mocks, wrong symbols, wrong strategy>"
  }
}
EOF
```

Only suggest retry when `latest.commit_sha`, the local HEAD captured in Phase 1, and the upstream sha captured in Phase 1 all match. Otherwise retry regenerates against a stale sha. Note: the upstream sha comes from `git rev-parse @{u}` which reflects the last `git fetch` — if the user has an unusually stale local, they may want to run `git fetch` first for an accurate check.

---

## Phase 5 — Apply selected scenarios

`tusk unit get-diffs` returns file-level unified diffs — it cannot do hunk-level filtering. Strategy depends on the selection shape:

**Adopting whole files** (all scenarios in the run, or all scenarios in a specific file set):

```bash
# All files:
tusk unit get-diffs <run-id> | jq -r '.files[].diff' | git apply

# Filtered by file path:
tusk unit get-diffs <run-id> | jq -r '.files[] | select(.file_path | IN("path/one","path/two")) | .diff' | git apply
```

**Adopting a subset of scenarios within a single file** — fall back to per-scenario fetches:

```bash
tusk unit get-scenario --run-id <run-id> --scenario-id <scenario-id>
```

Response contains `test_code` and `test_file_path`. For each selected scenario: read `test_file_path` (create if needed), insert `test_code` alongside existing tests, and pull in any imports/helpers from the scenario's diff preamble so partial adoption doesn't silently break on missing setup.

**Verify**: run the applied tests locally. If they fail, investigate before reverting — the assertion may be valid and expose a real bug. Commit and push when green.

---

## Phase 6 — Send feedback

Send feedback for every evaluated scenario (both adopted and rejected) unless the user explicitly declined everything with no reason to record.

```bash
tusk unit feedback --run-id <run-id> --file - <<'EOF'
{
  "scenarios": [
    {
      "scenario_id": "<uuid>",
      "positive_feedback": ["covers_critical_path"],
      "applied_locally": true
    },
    {
      "scenario_id": "<uuid>",
      "negative_feedback": ["no_value"],
      "comment": "Too many tests for a trivial function",
      "applied_locally": false
    }
  ]
}
EOF
```

The payload also supports a top-level `run_feedback.comment` for run-level guidance (used in the Phase 4 retry-with-feedback suggestion). You can combine per-scenario feedback and run-level feedback in the same payload.

**Valid `positive_feedback` codes**: `covers_critical_path`, `valid_edge_case`, `caught_a_bug`, `other`.

**Valid `negative_feedback` codes**: `incorrect_business_assumption`, `duplicates_existing_test`, `no_value`, `incorrect_assertion`, `poor_coding_practice`, `other`.

Include a short `comment` when the rejection reason is non-obvious. Adopted scenarios should get at least one `positive_feedback` code.

---

## Error handling

- **`tusk` not installed** → install from https://docs.usetusk.ai/onboarding and stop.
- **Auth error** (mentions `tusk auth login`) → run `tusk auth login` and stop.
- **5xx** → Tusk service error. Retry; if it persists, contact support@usetusk.ai.
- **Other unexpected errors** → surface the raw message and let the user decide.
