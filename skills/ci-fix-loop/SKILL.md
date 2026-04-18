---
name: ci-fix-loop
description: Autonomous CI fix loop with background monitoring and retry logic. Runs up to 10 fix-commit-push-wait cycles until CI passes or max retries reached.
---

# CI Fix Loop

## Role

Drive an autonomous CI-repair cycle: analyze the latest failure, apply fixes, commit, push, wait for the new CI run, and loop until CI passes or the retry budget is exhausted. The point is to free the user from babysitting a CI run that's failing on fixable errors (formatting, obvious lint, easy test fixes).

Invoked by `/fix-ci --loop`, `/fix-ci --auto`, or `/fix-ci --swarm`.

## Priorities

Correct fix (no suppressed signal) > Forward progress (commit and push every iteration) > Termination (bail when progress stalls)

## Invariants

- **Don't run on `main` / `master`.** Autonomous loops that push to a protected branch are dangerous even when the fixes are right. Abort and suggest a hotfix branch.
- **Stash before starting.** Preserve the user's in-progress work; restore on termination regardless of outcome.
- **Bail when progress stalls.** If the same errors reappear across two consecutive attempts, the loop isn't converging — human intervention beats burning more CI budget.
- **Budget caps are hard.** Maximum 10 attempts; maximum 30 minutes per CI run; maximum 2 minutes waiting for a new run to start after push. Each limit exists because the failure mode beyond it is worse than stopping here.
- **Never `--no-verify` a commit.** Local hooks catch classes of regressions that CI won't; skipping them lets bugs slip into the loop itself.

## Flag parsing

Before anything else, extract flags from `$ARGUMENTS`:
- `--loop` or `--auto` → standard autonomous mode.
- `--swarm` → parallel fix mode (see below). Remember this choice — it affects how fixes are applied in each attempt.

Strip consumed flags; anything remaining is contextual (e.g., a PR number).

## Configuration

| Setting | Value | Reason |
|---|---|---|
| `max_attempts` | 10 | Beyond this, compounding failures usually mean the root cause isn't code |
| `poll_interval` | 60s | CI status doesn't change faster than this anyway |
| `ci_start_timeout` | 120s | If CI hasn't started in 2 min, the workflow's likely misconfigured |
| `ci_run_timeout` | 1800s (30 min) | Longer runs usually indicate infra flake, not a fixable error |

## Initialization

```bash
BRANCH=$(git branch --show-current)
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null || echo "unknown")
```

Guard on protected branches:
```bash
if [[ "$BRANCH" == "main" || "$BRANCH" == "master" ]]; then
  echo "Cannot run autonomous fixes on $BRANCH. Create a feature branch: git checkout -b fix/ci-errors"
  # abort
fi
```

Stash uncommitted changes:
```bash
if [[ -n $(git status --porcelain) ]]; then
  git stash push -m "pre-ci-fix-loop-$(date +%Y%m%d_%H%M%S)"
fi
```

Start the loop with `attempt=1`, `last_errors=[]`, `history=[]`, `consecutive_same_errors=0`.

## Iteration

Each attempt follows the same shape. The sequence matters — commit must come after fix, push must come after commit, monitor must come after push — but within those boundaries the model picks the tactics. Don't over-narrate each step.

### Fetch and analyze

Pull the latest failed run on this branch, fetch logs for every failed job, and hand them to the `ci-log-analyzer` agent. The agent returns a structured error list (category, file, line, message). Trust that output — re-parsing logs wastes context.

```bash
RUN_ID=$(gh run list --branch "$BRANCH" --limit 5 --json databaseId,conclusion \
  --jq '[.[] | select(.conclusion == "failure")][0].databaseId')
FAILED_JOBS=$(gh run view $RUN_ID --json jobs --jq '.jobs[] | select(.conclusion == "failure") | .databaseId')
for JOB_ID in $FAILED_JOBS; do
  gh api repos/${REPO}/actions/jobs/${JOB_ID}/logs > /tmp/ci-logs-${JOB_ID}.txt 2>/dev/null || true
done
```

If no failed runs exist, CI might already be passing — skip to monitoring to confirm.

### Termination check (no progress)

Compare the current error list to `last_errors`. If they're identical after at least one fix attempt, increment `consecutive_same_errors`. At 2, exit the loop — the fixes aren't landing or the errors are beyond what this skill can handle. Report the persistent errors in the final summary; human intervention is the right move.

### Apply fixes

Default path: single `ci-error-fixer` agent handles the whole error list sequentially. Appropriate when errors are in one or two files, or when swarm mode wasn't requested.

**Swarm path** — triggered when `--swarm` was set *and* errors span 2+ distinct files:

Split the error list into up to 4 partitions by file (grouping by directory proximity if there are more than 4 files). Non-file-specific errors (dependency resolution, config, flaky tests without file association) stay with the lead for sequential handling.

Create the team with `TeamCreate`: name `fix-ci-{YYYYMMDD-HHMMSS}`, description `CI Fix Attempt {attempt}`. If `TeamCreate` is unavailable (experimental flag off), fall back to the single-agent path — surface the fallback in logs but don't abort the attempt.

Spawn one teammate per partition via `Task` with `team_name` and `subagent_type: "general-purpose"`. Teammates can't see this conversation — embed partition details as literal text.

<example name="Fix-loop teammate prompt">
You are fixing CI errors for a specific set of files as part of a parallel fix team.

YOUR FILE PARTITION:
{literal list of file paths}

ERRORS TO FIX:
{literal error list for these files — type, file, line, message}

FILE CONTENTS:
{read each file and include its full contents here}

Apply targeted fixes per error type: lint with the formatter or linter, type errors with correct annotations or mismatch fixes, test failures with corrected logic or implementation bugs, build errors with import/dependency fixes.

Constraints (parallel-team context):
- Edit only your assigned files. The lead handles staging and commits to avoid conflicts.
- Don't run tests or builds — other teammates are editing concurrently.
- Communicate via `SendMessage` only; `AskUserQuestion` isn't available in team context.
- Make minimal, targeted fixes. Over-editing introduces new errors and complicates the lead's review.
- Read files completely, no limit/offset.

When every error in your partition is fixed:
1. `SendMessage` `FIX COMPLETE`.
2. Wait for `shutdown_request`.
</example>

Wait for every teammate's `FIX COMPLETE`, up to 10 minutes per teammate from spawn. On timeout, proceed with available fixes and note it in the history. After teammates complete (or time out), the lead stages all changes in one commit (below) rather than per-teammate.

**Cleanup invariant** — regardless of swarm success or failure, `SendMessage` `type: "shutdown_request"` to each teammate, wait briefly, then `TeamDelete`. Skipping leaks team slots.

### Commit and push

One commit per iteration — captures the full fix for this attempt in a single reviewable unit:

```bash
git add .
git commit -m "fix(ci): automated fix attempt ${attempt}

Errors addressed:
- ${error_summary_list}

Attempt ${attempt} of ${max_attempts} (ci-fix-loop${SWARM_SUFFIX})"

PUSH_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)
git push origin ${BRANCH}
```

`PUSH_TIME` is used below to detect the new run (distinguishing it from the run currently being analyzed). In swarm mode the lead owns this step — teammates never run git commands, which keeps coordination simple.

### Wait for the new run to start

Use the `Monitor` tool (no LLM tokens burned on polling) to watch for a run created after `PUSH_TIME`, capped at `ci_start_timeout`:

```
Monitor:
  description: "CI run start detection on ${BRANCH}"
  timeout_ms: 120000
  persistent: false
  command: |
    while true; do
      RUN_JSON=$(gh run list --branch "$BRANCH" --limit 1 --json databaseId,status,createdAt 2>&1) || {
        echo "WARN: gh run list failed, retrying..." >&2
        sleep 5; continue
      }
      CREATED=$(echo "$RUN_JSON" | jq -r '.[0].createdAt // empty')
      if [ -n "$CREATED" ] && [[ "$CREATED" > "$PUSH_TIME" ]]; then
        RUN_ID=$(echo "$RUN_JSON" | jq -r '.[0].databaseId')
        echo "STARTED|$RUN_ID"
        exit 0
      fi
      sleep 5
    done
```

Notifications:
- `STARTED|{RUN_ID}` → capture as `NEW_RUN_ID`, proceed to monitoring.
- Monitor timeout (no notification within 120s) → warn `"No CI run started after 120s — check if workflows are enabled for this branch"` and proceed with `TIMEOUT`.

### Monitor to terminal state

Run the `Monitor` tool again, filtered by `NEW_RUN_ID` when known (avoids race conditions with concurrent runs on the same branch):

```
Monitor:
  description: "CI run ${NEW_RUN_ID} on ${BRANCH}"
  timeout_ms: 1800000
  persistent: false
  command: |
    while true; do
      if [ -n "$NEW_RUN_ID" ]; then
        RESULT=$(gh run view "$NEW_RUN_ID" --json databaseId,status,conclusion 2>&1) || {
          echo "WARN: gh run view failed, retrying..." >&2
          sleep 60; continue
        }
        STATUS=$(echo "$RESULT" | jq -r '.status // "unknown"')
        CONCLUSION=$(echo "$RESULT" | jq -r '.conclusion // "null"')
        RUN_ID="$NEW_RUN_ID"
      else
        RESULT=$(gh run list --branch "$BRANCH" --limit 1 --json databaseId,status,conclusion 2>&1) || {
          echo "WARN: gh run list failed, retrying..." >&2
          sleep 60; continue
        }
        STATUS=$(echo "$RESULT" | jq -r '.[0].status // "unknown"')
        CONCLUSION=$(echo "$RESULT" | jq -r '.[0].conclusion // "null"')
        RUN_ID=$(echo "$RESULT" | jq -r '.[0].databaseId // "unknown"')
      fi
      case "$STATUS" in
        completed)
          case "$CONCLUSION" in
            success)    echo "SUCCESS|$RUN_ID"; exit 0 ;;
            failure)    echo "FAILURE|$RUN_ID"; exit 1 ;;
            cancelled)  echo "CANCELLED|$RUN_ID"; exit 2 ;;
            skipped)    echo "SUCCESS|$RUN_ID"; exit 0 ;;
            *)          echo "FAILURE|$RUN_ID"; exit 1 ;;
          esac ;;
        requested|waiting|queued|pending|in_progress)
          ;; # still running, continue polling
        action_required)
          echo "ACTION_REQUIRED|$RUN_ID"; exit 3 ;;
      esac
      sleep 60
    done
```

Handle the result:
- **SUCCESS** — exit the loop, report success.
- **FAILURE** — record history, `last_errors = current_errors`, `attempt += 1`, back to `Fetch and analyze`.
- **CANCELLED** — warn and exit. Someone (or something) cancelled the run externally; continuing would loop indefinitely.
- **ACTION_REQUIRED** — warn the user that manual approval is needed and exit.
- **TIMEOUT** (30 min) — report and suggest `gh run watch`; offer continue-waiting vs. abort — long CI runs usually indicate infrastructure issues, not fixable errors.

### Record

Append to history:
```
{attempt, errors_found, errors_fixed, errors_flagged, run_id, result, duration}
```

## Final report

On exit — success, failure, or abort — produce:

```
CI Fix Loop Complete

Result: SUCCESS | FAILURE | ABORTED after {attempts} attempt(s)
Total time: {duration}
Commits created: {count}
Errors fixed: {total}

History:
  Attempt 1: Found 5 errors, fixed 5 → FAILURE (5 new errors surfaced after first fix)
  Attempt 2: Found 5 errors, fixed 3 → SUCCESS
  ...
```

**On failure** — surface what's left so the human knows where to pick up:
```
Remaining issues (need manual intervention):
  - src/x.ts:42 — type mismatch on return value (error persisted across 2 attempts)

Next:
  1. Review remaining errors above.
  2. Inspect logs: gh run view {last_run_id} --log-failed
  3. Fix manually and push.
```

**On success** — report the commit trail so the user can squash or review:
```
CI is now passing.

Next:
  1. Review commits: git log --oneline -{commit_count}
  2. Squash if desired: git rebase -i HEAD~{commit_count}
  3. Open PR: /github:create-pr
```

## Error handling

- **Network / `gh` flakes** — retry the command up to 3 times with 5s backoff before treating as persistent; abort the loop if still failing.
- **Push rejection** (upstream advanced) — don't try to resolve automatically; tell the user `"Upstream changes detected. Pull and retry: git pull --rebase && /fix-ci --loop"`.
- **CI start timeout** (no run within 2 min) — check workflow configuration; continue to monitor just in case a delayed run appears.
- **CI run timeout** (30 min exceeded) — offer continue-waiting vs. abort; don't silently extend.
- **Team creation fails** (swarm only) — fall back to single-agent fix for that iteration; keep going.

## Safety recap

The invariants section at the top is load-bearing, but the mechanics also matter:
- Branch protection prevents protected-branch loops.
- Stash preserves uncommitted work.
- `consecutive_same_errors >= 2` aborts loops that aren't converging.
- Commit log lets the user squash or revert cleanly.
- In swarm mode, teammates edit files only — they don't run git, builds, or tests. That keeps coordination simple and prevents teammates from clobbering each other's work.
- CI monitoring uses the `Monitor` tool rather than an LLM polling agent — zero token cost for the wait loop.
