---
description: Auto-detect, analyze, and fix CI/CD failures on any branch
---

# Fix CI Failures

## Role

Find why CI is failing on a branch or PR and fix the underlying code so CI passes. Delegate log parsing to the `ci-log-analyzer` agent and fix application to the `ci-error-fixer` agent — trust their output rather than re-parsing logs in this command.

## Priorities

Root-cause fix (not a workaround) > Minimal blast radius (one commit per coherent fix) > Fast turnaround

## Invariants

- **Don't disable a test or lint rule to make CI green.** Those are signal. If a test fails, either the test is wrong (fix the test) or the code is wrong (fix the code). Suppressing the signal hides the bug.
- **No `--no-verify` on commits.** Pre-commit hooks catch regressions locally before CI even runs; skipping them moves failures downstream.
- **Branch protection holds.** Never run this workflow on `main` / `master` without explicit user approval. The safer path is a hotfix branch.
- **Preserve uncommitted work.** If the tree is dirty, stash it before applying fixes, then restore.

## Usage

```bash
/fix-ci              # current branch (single iteration)
/fix-ci 123          # PR number
/fix-ci https://...  # PR URL
/fix-ci --loop       # autonomous loop (up to 10 retries), delegates to ci-fix-loop skill
/fix-ci --auto       # alias for --loop
/fix-ci 123 --loop   # loop mode targeting a specific PR
```

## Routing

`--loop` or `--auto` in `$ARGUMENTS` → delegate to the `ci-fix-loop` skill. That skill owns the full autonomous cycle (analyze → fix → commit → push → wait on CI → repeat, up to 10 iterations), and handles background monitoring and per-iteration safety. This command should hand off cleanly — don't duplicate the loop logic here.

```bash
ARGS="$ARGUMENTS"
if [[ "$ARGS" == *"--loop"* ]] || [[ "$ARGS" == *"--auto"* ]]; then
  PR_NUM=$(echo "$ARGS" | grep -oE '^[0-9]+' || echo "")
  # Invoke ci-fix-loop skill with $PR_NUM; return.
fi
```

Without those flags, run the single-iteration workflow below.

## Single-iteration workflow

### Detect context

Figure out what CI pipeline this branch or PR is actually using. The detection order prioritizes PRs because they carry richer metadata than raw workflow runs:

```bash
CURRENT_BRANCH=$(git branch --show-current)
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

1. User passed a PR number or URL → use that directly.
2. Feature branch with an open PR → use PR checks (`gh pr status`, `gh pr view {PR} --json statusCheckRollup`).
3. Feature branch without a PR → fall back to recent workflow runs (`gh run list --branch`).
4. `main` / `master` → prompt before doing anything (see invariants); if approved, use workflow runs.

### Fetch the failed logs

Once the context is resolved, pull logs for every failed job. Save to `/tmp/ci-logs-{JOB_ID}.txt` so the log-analyzer agent can read them from disk without re-fetching:

```bash
# PR path
gh pr view {PR} --json statusCheckRollup | jq '.statusCheckRollup[] | select(.conclusion == "FAILURE")'
gh api repos/${REPO}/actions/jobs/{JOB_ID}/logs > /tmp/ci-logs-{JOB_ID}.txt

# Run path
RUN_ID=$(gh run list --branch "$CURRENT_BRANCH" --limit 5 --json databaseId,conclusion --jq '[.[] | select(.conclusion == "failure")][0].databaseId')
gh run view $RUN_ID --json jobs --jq '.jobs[] | select(.conclusion == "failure")'
gh api repos/${REPO}/actions/jobs/{JOB_ID}/logs > /tmp/ci-logs-{JOB_ID}.txt
```

### Find stage — parse errors via `ci-log-analyzer`

Launch the `ci-log-analyzer` agent with the log paths. The agent categorizes errors (lint / test / type / build), extracts file paths and line numbers, and returns a structured list. Trust that output — if the agent thinks an error is `type-error` at `src/x.ts:42`, don't re-read the logs to second-guess.

The agent covers common patterns: Ruff/Prettier/Biome format diffs, test failures (pytest `FAILED`, Jest `● FAIL`, Go `--- FAIL`), type errors (mypy, tsc, Flow), build errors (`SyntaxError`, `ImportError`, missing deps).

### Filter stage — decide what to fix this iteration

From the full error list, pick a coherent subset to address in a single commit. Rules of thumb:

- Fix every error in a single category before mixing (all lint first, then tests, then types) — one commit per category reads cleaner in git history.
- If one error is a root cause for others (e.g., a missing import cascades into multiple test failures), fix the root and let the follow-ups resolve themselves.
- If the error list has > 20 items across multiple categories, surface the plan to the user and ask which to prioritize rather than attempting everything at once.

Out of this iteration's scope: flakes that weren't reproduced, infrastructure issues (GitHub Actions outages, runner pool), errors in paths the current branch didn't touch. Note these in the summary rather than silently ignoring.

### Apply fixes via `ci-error-fixer`

Launch the `ci-error-fixer` agent with the filtered error list. The agent reads the affected files, applies fixes appropriate to each error type, and reports diffs. Trust the agent's output — it's the specialist for this flow.

### Summarize

```
Fixed {N} issues:
  • Lint: {file1} — {what changed}
  • Test: {file2} — {what changed}
  • Type: {file3} — {what changed}

Out of scope this run:
  • Flake at tests/x.spec.ts (not reproducible locally)

Next:
  1. Review: git diff
  2. Commit: git add {files} && git commit -m "fix: CI {category}"
  3. Push: git push
```

Don't auto-commit or auto-push in single-iteration mode — the user needs to approve the diff first. Loop mode delegates that responsibility to the `ci-fix-loop` skill.

## Safety gates

Apply before any fix lands:

- **On `main` / `master`.** Prompt: `"You're on $CURRENT_BRANCH. Suggest hotfix branch: git checkout -b hotfix/ci-fixes"`. Ask before continuing — fixing CI on main directly is usually the wrong shape.
- **Uncommitted changes.** Warn and offer to stash: `git stash push -u -m "pre-fix-ci"`. Restore after (regardless of success/failure).
- **Ambiguous errors.** When the log-analyzer returns something that doesn't cleanly map to a fix strategy, flag it and ask the user. Guessing on an unfamiliar failure mode usually makes things worse.

## Error responses

```
Not authenticated       — Run: gh auth login
No failed checks found  — CI is passing or hasn't run yet.
Log fetch failed        — Suggest manual: gh run view $RUN_ID --log-failed
Agent fix failed        — Show agent output; suggest manual investigation.
```

## Notes

Requires `gh` CLI authenticated (`gh auth status`). Delegates parsing and fixing to specialized agents — this command's job is context detection, log fetching, and coordination.
