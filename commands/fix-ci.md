---
description: Auto-detect, analyze, and fix CI/CD failures on any branch
---

# Fix CI/CD Failures

Auto-detect, analyze, and fix CI/CD failures on any branch using GitHub CLI and specialized agents.

## Usage

```bash
/fix-ci              # Current branch
/fix-ci 123          # PR number
/fix-ci https://...  # PR URL
```

User provided: `$ARGUMENTS`

## Workflow

### Step 1: Detect Context

First, determine what CI context we're working with:

```bash
CURRENT_BRANCH=$(git branch --show-current)
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

**Priority order:**
1. If user provided PR number/URL: use that
2. Feature branch with PR: use PR checks
3. Feature branch without PR: use `gh run list`
4. Main/master: use `gh run list`

**Detection commands:**

For **main/master branch**, use workflow runs:
```bash
gh run list --branch "$CURRENT_BRANCH" --limit 5 --json databaseId,status,conclusion,name
```

For **feature branches**, check for PR first:
```bash
gh pr status --json number,title,url,headRefName
```

If no PR exists on feature branch, fall back to runs:
```bash
gh run list --branch "$CURRENT_BRANCH" --limit 5 --json databaseId,status,conclusion,name
```

### Step 2: Fetch Failed CI Logs

Once you've identified the context, fetch logs for failed checks.

**PR-based approach:**
```bash
# Get failed checks from PR
gh pr view {PR} --json statusCheckRollup | jq '.statusCheckRollup[] | select(.conclusion == "FAILURE")'

# Fetch job logs
gh api repos/${REPO}/actions/jobs/{JOB_ID}/logs > /tmp/ci-logs-{JOB_ID}.txt
```

**Run-based approach:**
```bash
# Get most recent failed run
RUN_ID=$(gh run list --branch "$CURRENT_BRANCH" --limit 5 --json databaseId,conclusion --jq '[.[] | select(.conclusion == "failure")][0].databaseId')

# Get failed jobs from run
gh run view $RUN_ID --json jobs --jq '.jobs[] | select(.conclusion == "failure")'

# Fetch job logs
gh api repos/${REPO}/actions/jobs/{JOB_ID}/logs > /tmp/ci-logs-{JOB_ID}.txt
```

### Step 3: Analyze Errors with CI Log Analyzer Agent

Launch the `ci-log-analyzer` agent to parse the logs and identify error patterns:

```bash
# Use Task tool to launch ci-log-analyzer agent
```

The agent will identify:
- **Ruff/Lint errors**: `Would reformat: {file}`, unused imports, etc.
- **Test failures**: `FAILED tests/...`, `AssertionError`, `ModuleNotFoundError`
- **Type errors**: `error: Incompatible types`, `Missing return statement`
- **Build errors**: `SyntaxError`, `ImportError`, missing dependencies

Agent returns structured error list with:
- Error type (lint/test/type/build)
- Affected file paths
- Error messages
- Line numbers (if available)

### Step 4: Apply Fixes with CI Error Fixer Agent

Launch the `ci-error-fixer` agent with the error list from Step 3:

```bash
# Use Task tool to launch ci-error-fixer agent with error context
```

The agent will:
1. Read affected files
2. Apply appropriate fixes based on error type:
   - **Ruff**: Run `uv run ruff format {file}` or apply formatting
   - **Tests**: Fix assertions, imports, test setup
   - **Types**: Add type hints, fix type mismatches
   - **Build**: Fix syntax, add missing imports
3. Show diffs for each change
4. Report completion status

### Step 5: Summary & Next Steps

Present a summary of all fixes applied:

```
✅ Fixed {N} issues:
  • Lint: {file1} - {description}
  • Test: {file2} - {description}
  • Type: {file3} - {description}

Next steps:
  1. Review changes: git diff
  2. Commit: git add . && git commit -m "fix: CI failures"
  3. Push: git push
```

## Safety Checks

**IMPORTANT: Run these checks before making any changes:**

1. **Main/master branch warning:**
   ```bash
   if [[ "$CURRENT_BRANCH" == "main" || "$CURRENT_BRANCH" == "master" ]]; then
     echo "⚠️  WARNING: You're on $CURRENT_BRANCH branch."
     echo "Suggest creating hotfix branch: git checkout -b hotfix/ci-fixes"
     # Ask user if they want to continue or create branch
   fi
   ```

2. **Uncommitted changes:**
   ```bash
   if [[ -n $(git status -s) ]]; then
     echo "⚠️  WARNING: You have uncommitted changes."
     echo "Consider stashing: git stash"
     # Ask user if they want to continue
   fi
   ```

3. **Unclear fixes:**
   - If error patterns are ambiguous or fixes are uncertain
   - Flag for manual review
   - Show the error and ask user how to proceed

4. **Multiple failures:**
   - If there are many failures across different categories
   - Ask user which to prioritize (lint/test/type/build)
   - Or ask if they want to fix all

## Error Handling

If any step fails:
- Show clear error message
- Suggest manual investigation steps
- Provide relevant gh CLI commands for debugging

## Notes

- Requires `gh` CLI to be installed and authenticated
- Uses specialized agents for complex parsing and fixing
- Always shows diffs before finalizing
- Never commits automatically without user confirmation
