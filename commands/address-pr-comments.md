---
description: Interactive or autonomous PR comment resolution with mandatory replies
---

# Address PR Comments Command

Comprehensive PR comment resolution workflow that addresses reviewer feedback and posts replies to GitHub for every comment.

User provided: `$ARGUMENTS`

## Workflow Overview

This command fetches all PR comments, processes them according to mode (interactive/autonomous), applies fixes where possible, and **posts a reply to GitHub for every comment** — ensuring no feedback is left unacknowledged.

## Phase 1: Preflight Checks and Comment Fetching

### 1.1 Preflight Checks

Before any processing, verify the environment is ready. These checks prevent partial failures mid-processing (e.g., committing without git identity, posting replies without auth, or accidentally staging uncommitted work alongside PR fixes).

1. **`gh auth status`** — The command posts replies to GitHub. Without authentication, it would process comments and apply fixes but fail silently when posting. Abort early with "Not authenticated. Run: gh auth login"
2. **`git status --porcelain`** — The command creates per-fix commits. If the working tree is dirty, those unrelated changes could get accidentally staged alongside PR fixes. Abort with "Working tree has uncommitted changes. Stash or commit changes first."
3. **`git config user.name` and `git config user.email`** — Required for creating commits. Without these, git will either fail or create commits with no author. Abort with instructions to configure.

Fail fast on any of these — proceeding would cause confusing partial failures later.

### 1.2 Extract PR Information

Parse `$ARGUMENTS` to extract PR number and determine repository context:

- If argument is a number (e.g., "123"): Use as PR number
- If argument is a GitHub URL (e.g., "https://github.com/owner/repo/pull/123"): Extract PR number
- If no argument: Abort with error "PR number or URL required"

Determine repository owner and name from current directory or URL.

### 1.3 Fetch All Comment Types

Fetch all three types of PR comments using GitHub API with pagination:

```bash
# Get current authenticated user (for reply deduplication)
CURRENT_USER=$(gh api user --jq '.login')

# 1. Review comments (file/line-specific comments)
gh api --paginate repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/comments > review_comments.json

# 2. Review summaries (top-level review comments)
gh api --paginate repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/reviews > review_summaries.json

# 3. Issue comments (general PR discussion)
gh api --paginate repos/{OWNER}/{REPO}/issues/{PR_NUMBER}/comments > issue_comments.json
```

### 1.4 Normalize Comment Structure

Parse and normalize all three types into a unified structure. Each comment should have:

```javascript
{
  id: number,                    // Comment ID for reply targeting
  type: string,                  // "review" | "review-summary" | "issue"
  path: string | null,           // File path (review comments only)
  line: number | null,           // Line number (review comments only)
  diff_hunk: string | null,      // Diff context (review comments only)
  body: string,                  // Comment text
  user: string,                  // Username of commenter
  created_at: string,            // Timestamp
  in_reply_to_id: number | null, // Parent comment ID (null if top-level)
  review_state: string | null    // Review state for summaries (APPROVED, CHANGES_REQUESTED, etc.)
}
```

**Type-specific normalization:**

1. **Review comments** (`type: "review"`):
   - Extract from `review_comments.json`
   - Has `path`, `line`, `diff_hunk`
   - May have `in_reply_to_id`

2. **Review summaries** (`type: "review-summary"`):
   - Extract from `review_summaries.json`
   - No file/line context
   - Has `state` field (APPROVED, CHANGES_REQUESTED, COMMENTED, etc.)
   - `body` may be empty/null

3. **Issue comments** (`type: "issue"`):
   - Extract from `issue_comments.json`
   - No file/line context
   - General discussion

### 1.5 Filter Comments

Apply filtering rules to determine which comments require processing:

**EXCLUDE:**
- Comments where `in_reply_to_id != null` (already a reply in a thread)
- Comments where `user` matches PR author's username
- Comments from bots (user contains "bot" or "[bot]")
- Comments with empty/whitespace-only `body`

**INCLUDE:**
- All other comments from review comments, review summaries, and issue comments
- Questions, suggestions, praise, requests — everything gets processed

**DO NOT filter out:**
- Questions (they need answers)
- Praise (they deserve acknowledgment)
- Out-of-scope comments (they need deferral responses)

### 1.6 Check for Existing Replies (Idempotency)

For each comment that passed filtering, check if the current user has already replied:

```bash
# For review comments (type: "review")
# Check if any reply in the comment thread is from CURRENT_USER
gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/comments/{COMMENT_ID} --jq '.user.login'
# Also check the thread by looking at all comments and filtering by in_reply_to_id

# For issue comments (type: "issue")
# Check if any subsequent issue comment quotes/references this one from CURRENT_USER

# For review summaries (type: "review-summary")
# Check if any issue comment from CURRENT_USER references the review ID
```

**Mark as `already_replied: true`** if an existing reply from current user is found. These comments will be skipped from reply posting but shown in the summary.

### 1.7 Confidence Scoring

Score each comment 0-100 based on how confidently you can apply the right fix without human judgment. This score drives autonomous mode filtering and prioritization in interactive mode.

**Judgment dimensions:**

- **Specificity**: Does the comment point to exact code and suggest a concrete change? A comment saying "use const instead of let on line 42" is near-certain; "consider refactoring" is not. Comments with `path`, `line`, and code examples score highest.
- **Clarity**: Is the intent unambiguous? Directive language ("please change", "must fix") is clearer than hedged language ("maybe consider", "could perhaps"). Questions without suggested changes score lowest.
- **Risk**: Is the fix low-risk (naming, formatting, documentation) or could it change behavior (logic, API contracts, test assertions)? Low-risk changes can be applied with higher confidence.
- **Authority**: Is the reviewer a maintainer with project context (`OWNER`, `COLLABORATOR`), or an outside contributor who may misunderstand the design? Maintainer feedback carries more weight.
- **Urgency**: Is this from a `CHANGES_REQUESTED` review (blocking merge) or a casual `COMMENTED` review? Blocking reviews deserve higher priority.

**Thresholds:**

- **80+**: Safe to auto-fix in autonomous mode — clear, specific, low-risk
- **60-79**: Present to user in interactive mode — reasonable but needs human judgment
- **Below 60**: Reply without fixing — too ambiguous to act on confidently

Use your judgment. These dimensions are guidelines for holistic assessment, not arithmetic. A comment from a maintainer saying "please rename this variable" might score 95; a vague comment from a contributor saying "this could be better" might score 30.

Store the score with each comment for filtering and reporting.

## Phase 2: Per-Comment Processing with Mandatory Replies

### 2.1 Detect Execution Mode

Auto-detect whether to run in interactive or autonomous mode:

```bash
# Check for explicit mode override in arguments
if [[ "$ARGUMENTS" =~ (--autonomous|--auto|auto) ]]; then
  MODE="autonomous"
elif [[ "$ARGUMENTS" =~ (--interactive|interactive) ]]; then
  MODE="interactive"
# Check for CI/CD environment
elif [[ -n "$CI" || -n "$GITHUB_ACTIONS" || -n "$JENKINS_URL" || -n "$GITLAB_CI" || -n "$CIRCLECI" || -n "$TRAVIS" ]]; then
  MODE="autonomous"
# Check if stdin is a TTY
elif [[ ! -t 0 ]]; then
  MODE="autonomous"
else
  MODE="interactive"
fi
```

### 2.2 Create Safety Checkpoint (Autonomous Mode Only)

**Before making any changes in autonomous mode**, create a rollback point:

```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
STASH_NAME="pre-autonomous-pr-${PR_NUMBER}-${TIMESTAMP}"

# Stash any uncommitted changes
if [ -n "$(git status --porcelain)" ]; then
  git stash push -u -m "$STASH_NAME"
  echo "✓ Uncommitted changes stashed as: $STASH_NAME"
fi

# Record current commit for reference
ROLLBACK_SHA=$(git rev-parse HEAD)
echo "✓ Safety checkpoint created at commit: $ROLLBACK_SHA"
```

### 2.3 Present Comments (Interactive Mode)

**IF MODE = "interactive":**

Display all comments (regardless of confidence score) in organized format:

```
Found {N} comments on PR #{NUMBER}: {TITLE}

Comments to Review:
─────────────────────────────────────────────────────────

1. [CODE] {file_path}:{line} (Score: 85) — @{reviewer}
   "{comment body excerpt}"
   Suggested action: {what needs to be done}

2. [STYLE] {file_path}:{line} (Score: 95) — @{reviewer}
   "{comment body excerpt}"
   Suggested action: {what needs to be done}

3. [QUESTION] {file_path}:{line} (Score: 40) — @{reviewer}
   "{comment body excerpt}"
   Suggested action: Provide explanation

4. [DOCS] No file specified (Score: 60) — @{reviewer}
   "{comment body excerpt}"
   Suggested action: Update documentation

─────────────────────────────────────────────────────────

Which items would you like me to address?
Options: "1,3,4" | "1-5" | "all" | "none"
```

**Use AskUserQuestion tool** to get user selection.

Parse user response and mark selected comments for processing.

### 2.4 Filter Comments (Autonomous Mode)

**IF MODE = "autonomous":**

Filter comments by confidence threshold (≥80) and display processing plan:

```
🤖 AUTONOMOUS MODE ACTIVE

Analyzing {N} total comments on PR #{NUMBER}: {TITLE}
Confidence threshold: 80/100

High-Confidence Items (will be addressed):
─────────────────────────────────────────────────────────

1. [STYLE] src/utils.ts:42 (Score: 95) — @maintainer
   Comment: "Please use const instead of let for maxRetries"
   Action: Change variable declaration → will fix + reply

2. [CODE] src/auth.ts:120 (Score: 90) — @owner
   Comment: "Add null check before accessing user.email"
   Action: Add safety check → will fix + reply

Below-Threshold Items (will reply without fix):
─────────────────────────────────────────────────────────

3. [CODE] src/api.ts:55 (Score: 65) — @contributor
   Comment: "Consider refactoring this for better performance"
   Action: Reply requesting clarification

4. [QUESTION] src/models.ts:78 (Score: 35) — @contributor
   Comment: "This approach might have issues"
   Action: Reply requesting specific concerns

─────────────────────────────────────────────────────────

Processing {X} high-confidence items + replying to all {N} comments...
```

### 2.5 Process Each Comment

For each comment (whether selected in interactive mode or high-confidence in autonomous mode), follow this workflow:

#### Category: `actionable/clear` (Score ≥ 80, has file+line)

**Goal:** Apply the reviewer's suggestion correctly, commit it atomically, and reply with the commit SHA.

**Key principles:**

- **Locate code by context, not line numbers.** The `diff_hunk` from the comment shows surrounding code context. Use this to find the right location because line numbers may have shifted from earlier fixes on the same file. If the context can't be found, treat as `missing-file`.
- **One commit per fix** so each fix is traceable and independently revertable. Stage only the file you changed (`git add {path}` — never `git add .` or `git add -A`, because that risks staging unrelated changes).
- **Verify before committing.** Check `git diff --quiet -- "{path}"` after applying the edit. If the file didn't actually change, the code already matches the suggestion — reply accordingly instead of creating an empty commit.
- **If the file is missing or renamed**, don't guess — categorize as `missing-file` and reply asking the reviewer to confirm relevance.

**Commit message format:** `fix: {description}` with body referencing reviewer and comment ID, because this creates an audit trail linking commits to specific review feedback:
```
fix: {brief description}

Addresses comment from @{reviewer}
Comment ID: {id}
```

Capture the commit SHA via `git rev-parse --short HEAD` and include it in the reply so the reviewer can click through to see exactly what changed.

Use your judgment on fix complexity. If the suggestion requires changes across multiple files or has behavioral implications you're unsure about, categorize as `actionable/unclear` and ask for clarification instead of guessing.

#### Category: `actionable/unclear` (Score 60-79, or unclear suggestion)

Reply asking for clarification. Show the reviewer you understood their comment by offering specific interpretations of what they might mean, rather than a generic "please clarify." This narrows the discussion and helps them respond quickly.

No commit — just reply.

#### Category: `not-actionable/question`

Read the relevant code (if the comment has a `path`, read that file for context) and answer the actual question. Explain *why* the code is written that way based on what you see. If you don't know, say so honestly and suggest who might.

No commit — just reply.

#### Category: `not-actionable/praise`

Brief acknowledgment. Don't over-elaborate — match the reviewer's energy.

No commit — just reply.

#### Category: `not-actionable/out-of-scope`

Acknowledge the value of the suggestion and explain why it's being deferred, not dismissed. If you can identify a follow-up context (future PR, separate issue), mention it.

No commit — just reply.

#### Category: `below-threshold` (Autonomous mode only, Score < 80)

Be honest about why you didn't auto-fix. Explain the specific reason the confidence was low (e.g., "suggestion lacks concrete implementation details" or "change could affect behavior in ways I can't verify"). The reviewer should understand what to do next.

No commit — just reply.

#### Category: `missing-file`

The file referenced in the comment doesn't exist. Reply noting it may have been moved or renamed, and ask the reviewer to confirm if the comment is still relevant.

No commit — just reply.

#### Interactive Mode: User Did Not Select

For comments the user chose not to address, reply noting the comment has been deferred to manual review.

No commit — just reply.

### Reply Quality Criteria

Every reply should be **contextual and useful** — not generic boilerplate. The reviewer should feel their specific feedback was understood, not that a bot processed it.

**Judgment criteria:**
- **Actionable fixes**: Reference the specific commit SHA. Briefly describe what changed so the reviewer doesn't have to click through.
- **Clarification requests**: Offer specific interpretations rather than "please clarify." Show you read the comment.
- **Questions**: Answer the actual question with code context. Don't dodge with "great question!"
- **Praise/acknowledgment**: Brief and genuine. One sentence is fine.
- **Deferrals**: Explain *why* it's deferred, not just *that* it's deferred.

Adapt tone to the reviewer's tone. Match their formality level. A casual "LGTM" deserves a casual acknowledgment, not a formal response.

### 2.6 Skip Already-Replied Comments

If `already_replied: true` (detected in Phase 1.6), skip reply posting but include in summary report.

## Phase 3: Reply Posting Logic

### 3.1 Safe Reply Body Construction

**CRITICAL: Prevent shell injection and malformed JSON.**

Always construct reply bodies using `jq` with `--arg` flag and pipe to `gh api --input -`:

```bash
# CORRECT METHOD (prevents injection):
jq -n --arg body "$REPLY_TEXT" '{body: $body}' | \
  gh api -X POST "repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/comments/{COMMENT_ID}/replies" --input -

# NEVER USE THIS (unsafe):
gh api -X POST "..." -f body="$REPLY_TEXT"
```

### 3.2 Reply to Review Comments (type: "review")

For comments with `type: "review"`:

```bash
ENDPOINT="repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}/comments/${COMMENT_ID}/replies"

jq -n --arg body "$REPLY" '{body: $body}' | \
  gh api -X POST "$ENDPOINT" --input -
```

### 3.3 Reply to Issue Comments (type: "issue")

For comments with `type: "issue"`:

1. **Format reply with quote** (truncate original to 200 chars):
   ```bash
   QUOTED_ORIGINAL=$(echo "$ORIGINAL_BODY" | head -c 200)
   if [ ${#ORIGINAL_BODY} -gt 200 ]; then
     QUOTED_ORIGINAL="${QUOTED_ORIGINAL}..."
   fi

   FORMATTED_REPLY=$(cat <<EOF
   > ${QUOTED_ORIGINAL}

   ${REPLY}
   EOF
   )
   ```

2. **Post as new issue comment**:
   ```bash
   ENDPOINT="repos/${OWNER}/${REPO}/issues/${PR_NUMBER}/comments"

   jq -n --arg body "$FORMATTED_REPLY" '{body: $body}' | \
     gh api -X POST "$ENDPOINT" --input -
   ```

### 3.4 Reply to Review Summaries (type: "review-summary")

For comments with `type: "review-summary"`:

1. **Format reply referencing the review**:
   ```bash
   # Handle empty/null body
   if [ -z "$ORIGINAL_BODY" ] || [ "$ORIGINAL_BODY" = "null" ]; then
     QUOTED_TEXT="No summary text provided — addressing based on review state (${REVIEW_STATE})."
   else
     QUOTED_ORIGINAL=$(echo "$ORIGINAL_BODY" | head -c 200)
     if [ ${#ORIGINAL_BODY} -gt 200 ]; then
       QUOTED_ORIGINAL="${QUOTED_ORIGINAL}..."
     fi
     QUOTED_TEXT="$QUOTED_ORIGINAL"
   fi

   FORMATTED_REPLY=$(cat <<EOF
   > From @${REVIEWER}'s review (${REVIEW_STATE}):
   > ${QUOTED_TEXT}

   ${REPLY}
   EOF
   )
   ```

2. **Post as issue comment** (review summaries don't have reply threads):
   ```bash
   ENDPOINT="repos/${OWNER}/${REPO}/issues/${PR_NUMBER}/comments"

   jq -n --arg body "$FORMATTED_REPLY" '{body: $body}' | \
     gh api -X POST "$ENDPOINT" --input -
   ```

### 3.5 Error Handling for Reply Posting

Handle API errors gracefully with retries:

```bash
# Retry logic with exponential backoff
MAX_RETRIES=3
RETRY_COUNT=0
BACKOFF=1

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
  HTTP_CODE=$(jq -n --arg body "$REPLY" '{body: $body}' | \
    gh api -X POST "$ENDPOINT" --input - -i 2>&1 | grep "HTTP" | awk '{print $2}')

  if [ "$HTTP_CODE" = "201" ] || [ "$HTTP_CODE" = "200" ]; then
    echo "✓ Reply posted successfully"
    break
  elif [ "$HTTP_CODE" = "403" ] || [ "$HTTP_CODE" = "429" ]; then
    # Rate limited or forbidden - check Retry-After header
    RETRY_AFTER=$(gh api -i ... | grep -i "retry-after" | awk '{print $2}')
    WAIT_TIME=${RETRY_AFTER:-$BACKOFF}
    echo "⏳ Rate limited, waiting ${WAIT_TIME}s before retry..."
    sleep $WAIT_TIME
    BACKOFF=$((BACKOFF * 2))
    RETRY_COUNT=$((RETRY_COUNT + 1))
  else
    echo "❌ Failed to post reply (HTTP $HTTP_CODE), skipping..."
    break
  fi
done

if [ $RETRY_COUNT -eq $MAX_RETRIES ]; then
  echo "❌ Max retries reached, reply not posted"
fi
```

## Phase 4: Summary Report

**Goal:** Give the user a clear picture of what happened — what was fixed, what was replied to, and what needs their attention.

### What to Include

- **Statistics**: Fixes applied, replies posted, items skipped (already replied)
- **Per-comment detail**: For each comment, show the action taken and reply status
- **Next steps**: What the user should do now (review changes, run tests, push)
- **Rollback instructions** (autonomous mode): How to undo commits, with a note that GitHub replies cannot be rolled back

### How to Structure

Separate comments into clear groups so the user can scan quickly:
1. **Comments Fixed + Replied** — code was changed, committed, and reply posted with SHA
2. **Comments Replied Without Fix** — questions answered, praise acknowledged, clarifications requested, or items deferred
3. **Already Replied (skipped)** — idempotency caught these; no action taken

For autonomous mode, include the rollback stash name and base commit SHA.

### Formatting Guidance

Adapt detail level to the volume of comments:
- **Few comments (< 10)**: Show each comment inline with its action and reply
- **Many comments (10+)**: Summarize by category with counts, list only the fixed items individually

Next steps should omit "post replies manually" (replies are already posted). Focus on: review the diff, run tests, push.

Note clearly that commits can be reverted but replies posted to GitHub are permanent.

## Error Handling

### PR Not Found
```
❌ Error: PR #{NUMBER} not found in {OWNER}/{REPO}
   Verify the PR number/URL and try again.
   Tip: Use `gh pr list` to see available PRs
```

### Not Authenticated
```
❌ Error: Not authenticated with GitHub CLI
   Run: gh auth login
```

### Working Tree Dirty
```
❌ Error: Working tree has uncommitted changes
   Stash them first: git stash
   Or commit them before addressing PR comments
```

### Git User Not Configured
```
❌ Error: Git user not configured
   Run:
     git config user.name 'Your Name'
     git config user.email 'your@email.com'
```

### No Comments Found
```
✅ No comments found on PR #{NUMBER}
   All reviewers are satisfied or comments have been addressed.
```

### All Comments Already Replied
```
✅ All comments on PR #{NUMBER} have already been replied to
   Nothing to process.
```

### File Not Found
```
⚠️  Warning: Comment references {file_path} which doesn't exist
    This comment may be outdated or refers to a renamed file.
    Reply posted: "This file appears to have been moved or renamed..."
```

### API Rate Limit
```
⏳ Rate limited by GitHub API
   Waiting {N} seconds before retry (attempt {X}/{MAX})...
```

### Reply Posting Failed
```
❌ Failed to post reply to comment {COMMENT_ID} (HTTP {CODE})
   Comment will be shown in summary but reply not posted.
   You may need to reply manually.
```

## Implementation Notes

### Comment Fetching
- Use `gh api --paginate` for all comment endpoints to handle repos with many comments
- Normalize all three comment types into unified structure for consistent processing
- Track `in_reply_to_id` to avoid processing existing replies

### Line Drift Safety
- **Always use `diff_hunk` context** for locating code, not just API `line` number
- API line numbers can become stale if files change after review
- Extract surrounding code from `diff_hunk` and use as anchor text for matching
- If context not found, treat as missing/renamed file

### Reply Deduplication (Idempotency)
- Before processing, check if current user already replied to each comment
- For review comments: check reply threads via `in_reply_to_id`
- For issue/review-summary comments: scan issue comment history
- Mark as `already_replied: true` and skip posting (but show in summary)

### Commit Strategy
- **Per-fix commits** (not batched): Each actionable comment gets its own commit
- Commit message format: `fix: {description}\n\nAddresses comment from @{reviewer}\nComment ID: {id}`
- Stage **only the specific file** changed: `git add {path}` (never `git add .`)
- Capture commit SHA for reply: `git rev-parse --short HEAD`

### Reply Safety
- **Always use `jq -n --arg body "$REPLY" '{body: $body}' | gh api --input -`**
- This prevents shell injection and handles special characters correctly
- Never use `-f body="..."` with complex/untrusted text

### Confidence Scoring
- Used for autonomous mode filtering (≥80 = auto-address) and interactive mode prioritization
- Holistic assessment across specificity, clarity, risk, authority, and urgency
- Store with each comment for reporting; scoring is heuristic — use judgment for edge cases

### Mode Detection
- Explicit flags (`--autonomous`, `--interactive`) override auto-detection
- CI/CD environments auto-select autonomous mode
- Non-TTY stdin auto-selects autonomous mode
- Default to interactive if no signals detected

### Safety in Autonomous Mode
- Create git stash before any changes for easy rollback
- Only auto-fix high-confidence items (score ≥80)
- **Never auto-commit in bulk or auto-push** — leave that to user
- Provide clear rollback instructions in summary
- Note that replies posted to GitHub cannot be rolled back

## Usage Examples

**Interactive mode (default in terminal):**
```bash
/address-pr-comments 123
/address-pr-comments https://github.com/owner/repo/pull/456
```

**Autonomous mode (auto-detected in CI or forced):**
```bash
/address-pr-comments 123 --autonomous
/address-pr-comments 123 auto

# In GitHub Actions workflow:
- name: Address PR Comments
  run: claude /address-pr-comments ${{ github.event.pull_request.number }}
  # Automatically runs in autonomous mode due to CI environment
```

**Force interactive mode:**
```bash
/address-pr-comments 123 --interactive
```

## Design Philosophy

### Every Comment Gets a Reply

This command ensures **no reviewer feedback is left unacknowledged**. Every comment receives a reply posted to GitHub, whether it's:
- A fix with commit SHA
- A request for clarification
- An acknowledgment of praise
- A deferral to manual review

This creates clear communication and prevents reviewers from wondering if their feedback was seen.

### Line-Drift Resilience

By using `diff_hunk` context for code location instead of relying solely on API `line` numbers, this command handles cases where files have changed since the review was posted.

### Safe Autonomous Operation

Autonomous mode is designed for CI/CD environments where no human is present to approve changes. It:
- Only auto-fixes high-confidence, low-risk items (score ≥80)
- Creates rollback checkpoints before changes
- Posts replies even for items it doesn't fix
- Provides detailed audit trail in summary report

### Per-Fix Commits

Each code fix gets its own commit with a descriptive message referencing the reviewer and comment ID. This makes it easy to:
- Review changes individually
- Cherry-pick or revert specific fixes
- Track which commit addressed which comment
- Generate meaningful git history

### Idempotency

Running the command multiple times is safe:
- Already-replied comments are detected and skipped
- No duplicate replies are posted
- Existing commits are not re-created
- Summary report clearly shows what was skipped vs. processed
