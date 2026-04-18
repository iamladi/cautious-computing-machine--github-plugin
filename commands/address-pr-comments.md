---
description: Interactive or autonomous PR comment resolution with mandatory replies
---

# Address PR Comments

## Role

Fetch every comment on a pull request, decide what to do with each one, apply fixes where it's safe to, and post a reply to every comment — no reviewer feedback leaves unanswered. Works in two modes: interactive (you approve each action before it runs) and autonomous (safe auto-fixes run unattended in CI).

## Priorities

Reviewer acknowledgment (every comment replied) > Fix safety (low-risk auto-fixes only) > Throughput

## Invariants

These hold regardless of mode — they're the properties that make this command trustworthy:

- **Every filtered comment receives one reply.** A fix without a reply is invisible to the reviewer; a reply without a fix at least closes the loop. Never skip the reply.
- **Idempotent reruns.** If the current user has already replied to a comment, skip it — duplicate replies confuse the thread.
- **One fix, one commit.** Stage only the files the fix touches (`git add {path}`), never `git add .` or `-A`. A bulk commit loses the audit trail and risks sweeping in unrelated uncommitted work.
- **Posted replies are permanent.** Commits can be reverted; GitHub replies cannot. Surface this in the summary for autonomous runs so the user knows what's reversible.
- **Reply body construction uses `jq --arg` piped to `gh api --input -`.** This prevents shell injection and JSON malformation from comment bodies containing `"`, `$`, or backticks. See **Reply posting** below.

## Preflight

Run these checks before fetching anything — failing partway through is worse than failing immediately. Each guards against a specific failure class:

- `gh auth status` — without auth, the command would fetch, process, and fix, then fail silently when posting. Abort with `"Not authenticated. Run: gh auth login"`.
- `git status --porcelain` — if the working tree has uncommitted changes, per-fix commits risk staging them alongside the fix. Abort with `"Working tree has uncommitted changes. Stash or commit first."`.
- `git config user.name` / `user.email` — required for commits. Abort with configuration instructions.

Parse `$ARGUMENTS` for the PR. Accept a PR number (`123`), a full URL (`https://github.com/owner/repo/pull/123`), or extract from context. If no PR identifier resolves, abort with `"PR number or URL required"`.

## Fetch and normalize

Pull all three comment surfaces with pagination:

```bash
CURRENT_USER=$(gh api user --jq '.login')
gh api --paginate repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/comments  > review_comments.json
gh api --paginate repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/reviews   > review_summaries.json
gh api --paginate repos/{OWNER}/{REPO}/issues/{PR_NUMBER}/comments > issue_comments.json
```

Normalize every comment into a unified shape — downstream logic shouldn't care which surface it came from:

```javascript
{
  id: number,
  type: "review" | "review-summary" | "issue",
  path: string | null,
  line: number | null,
  diff_hunk: string | null,
  body: string,
  user: string,
  created_at: string,
  in_reply_to_id: number | null,
  review_state: string | null  // summaries only: APPROVED, CHANGES_REQUESTED, etc.
}
```

## Filter and check for prior replies

Exclude:
- Comments already in a reply thread (`in_reply_to_id != null`) — those are conversation, not new feedback.
- Comments from the PR author themselves.
- Bot comments (user contains `bot` or `[bot]`).
- Empty or whitespace-only bodies.

Keep everything else — questions, suggestions, praise, out-of-scope requests. All of them get processed because the goal is "no feedback unacknowledged."

For every kept comment, check if the current user already replied. For review comments, walk the reply thread by `in_reply_to_id`. For issue comments and review summaries, scan later issue comments from `CURRENT_USER` that reference the original. Mark `already_replied: true` on matches — these get shown in the summary but skip the reply step. Idempotency matters because this command is often re-run in CI.

## Confidence scoring

Score each comment 0–100 by how confident you are that the right fix is unambiguous. Autonomous mode uses this to decide what to auto-fix; interactive mode uses it to sort.

Judge holistically across these dimensions, not arithmetically:

- **Specificity.** A comment with `path`, `line`, a code suggestion, and a clear directive scores high. `"consider refactoring"` with no code scores low.
- **Clarity.** Directive language (`"please change X to Y"`) is clearer than hedged (`"maybe consider"`). Questions without suggested changes score lowest.
- **Risk.** Naming, formatting, comment edits are low-risk. Logic changes, API contract edits, test assertion changes can shift behavior — lower the confidence unless the suggestion is extremely specific.
- **Authority.** Maintainer/owner comments carry more weight than outside contributors who may misunderstand the design. Not a hard rule; use judgment.
- **Urgency.** `CHANGES_REQUESTED` reviews block merge. Treat them as higher priority than casual `COMMENTED` reviews.

Thresholds for autonomous mode:
- **≥ 80** — safe to auto-fix. Clear, specific, low-risk.
- **60–79** — reply with clarification request. Ambiguous enough that guessing costs more than asking.
- **< 60** — reply acknowledging, but don't act.

These are defaults. A maintainer's `"please rename this variable"` might score 95; a vague contributor comment might score 30 — the score is a summary of judgment, not a formula.

## Mode detection

Auto-detect the mode; explicit flags win:

- `--autonomous` / `--auto` / `auto` in arguments → autonomous.
- `--interactive` / `interactive` in arguments → interactive.
- `$CI`, `$GITHUB_ACTIONS`, `$JENKINS_URL`, `$GITLAB_CI`, `$CIRCLECI`, `$TRAVIS` set → autonomous.
- Non-TTY stdin (`! [ -t 0 ]`) → autonomous.
- Default → interactive.

Autonomous mode before any edits, create a rollback checkpoint:

```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
STASH_NAME="pre-autonomous-pr-${PR_NUMBER}-${TIMESTAMP}"
[ -n "$(git status --porcelain)" ] && git stash push -u -m "$STASH_NAME"
ROLLBACK_SHA=$(git rev-parse HEAD)
```

Include both `$STASH_NAME` and `$ROLLBACK_SHA` in the final summary so rollback is a copy-paste.

## Present comments (interactive mode)

Show every non-already-replied comment with its confidence score, sorted by priority. Use `AskUserQuestion` for selection (`"1,3,4"`, `"1-5"`, `"all"`, `"none"`). Display format:

```
Found {N} comments on PR #{NUMBER}: {TITLE}

1. [CODE] src/auth.ts:120 (Score: 90) — @owner
   "Add null check before accessing user.email"
   Suggested action: Add safety check

2. [QUESTION] src/api.ts:55 (Score: 40) — @contributor
   "Is this the right approach?"
   Suggested action: Answer the question inline

[... etc]

Which items would you like me to address?
```

Only the user's selections get fixes. Everything else still gets a reply (deferral explanation) — the invariant holds.

## Filter comments (autonomous mode)

Display what will happen so the run is auditable. Items at ≥ 80 get fixes + replies; items below get replies only with honest reasons for not auto-fixing (e.g., `"suggestion lacks concrete implementation details"`, not a generic apology).

## Per-comment processing

Categorize each selected comment, then handle by category. Each category's reply reflects the actual action taken, not boilerplate:

### `actionable/clear` (score ≥ 80, has file and line)

Apply the fix, commit atomically, reply with the commit SHA.

- **Locate code by context, not line number.** Use the `diff_hunk` as an anchor — line numbers drift as the PR accumulates fixes from this same command. If the surrounding context can't be found in the file, treat as `missing-file` rather than guessing.
- **Verify before committing.** After the edit, `git diff --quiet -- "{path}"` tells you whether anything actually changed. If not, the code already matches the suggestion — reply accordingly instead of creating an empty commit.
- **One commit per fix.**
  ```bash
  git add {path}
  git commit -m "fix: {brief description}

  Addresses comment from @{reviewer}
  Comment ID: {id}"
  ```
- Capture the short SHA (`git rev-parse --short HEAD`) and include it in the reply. The reviewer should be able to click through to see exactly what changed.

If the fix requires edits across multiple files or has behavioral implications you can't verify, reclassify as `actionable/unclear` and ask instead of guessing.

### `actionable/unclear` (score 60–79, or suggestion ambiguous)

Reply with specific interpretations of what the reviewer might mean — `"I see two readings of this: (a) … (b) … — which did you mean?"`. Shows you read the comment and narrows the discussion. No commit.

### `not-actionable/question`

Read the relevant code (if the comment has a `path`) and answer the actual question with context. Explain *why* the code is the way it is, based on what's actually there. If you don't know, say so and suggest who would. No commit.

### `not-actionable/praise`

Brief acknowledgment. Match the reviewer's energy — a casual `"LGTM"` deserves a casual `"thanks"`, not a paragraph. No commit.

### `not-actionable/out-of-scope`

Acknowledge the value of the suggestion and explain the deferral. If you can point to a follow-up (future PR, separate issue), do so. Don't dismiss. No commit.

### `below-threshold` (autonomous mode only, score < 80)

Be honest about why you didn't auto-fix: `"suggestion lacks concrete implementation details"`, `"change could affect behavior in ways I can't verify"`. The reviewer should understand what action they can take next. No commit.

### `missing-file`

File referenced in the comment doesn't exist (or the `diff_hunk` context isn't found). Reply noting it may have been moved or renamed and ask the reviewer to confirm relevance. No commit.

### Interactive mode: not selected

Reply noting the comment has been deferred to manual review. No commit.

## Reply quality

Every reply should feel contextual, not like a bot processed it:

- Actionable fixes reference the specific commit SHA and briefly describe what changed.
- Clarification requests offer specific interpretations rather than generic `"please clarify"`.
- Question answers use code context — don't dodge with `"great question!"`.
- Praise gets one sentence.
- Deferrals explain *why* not just *that*.

Match the reviewer's tone and formality.

## Reply posting

**Security invariant: always construct reply bodies via `jq --arg` piped to `gh api --input -`.** Direct interpolation into `-f body="..."` breaks on comment bodies containing `"`, `$`, backticks, or newlines — worse, it's a shell-injection vector when the body is untrusted. This applies across every reply surface below.

```bash
# correct — injection-safe
jq -n --arg body "$REPLY" '{body: $body}' | gh api -X POST "$ENDPOINT" --input -

# wrong — unsafe
gh api -X POST "$ENDPOINT" -f body="$REPLY"
```

Endpoints vary by comment type:

- **Review comments** (`type: "review"`) — reply directly in the thread:
  ```bash
  ENDPOINT="repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}/comments/${COMMENT_ID}/replies"
  ```
- **Issue comments** (`type: "issue"`) — no reply threading, so quote the original (truncate bodies >200 chars to `...`) and post as a new issue comment:
  ```
  > {truncated original}

  {your reply}
  ```
  ```bash
  ENDPOINT="repos/${OWNER}/${REPO}/issues/${PR_NUMBER}/comments"
  ```
- **Review summaries** (`type: "review-summary"`) — no thread either. Quote the reviewer + review state, then post as an issue comment. If the original body is null (common for `APPROVED` reviews with no text), use `"No summary text provided — addressing based on review state ({state})."`.

### Retry on rate limit

GitHub throttles aggressive reply posting. Handle 429 and 403-with-rate-limit by reading `Retry-After` and backing off exponentially (1s, 2s, 4s), cap at 3 attempts:

```bash
MAX_RETRIES=3; RETRY_COUNT=0; BACKOFF=1
while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
  HTTP_CODE=$(jq -n --arg body "$REPLY" '{body: $body}' | \
    gh api -X POST "$ENDPOINT" --input - -i 2>&1 | grep "HTTP" | awk '{print $2}')
  if [ "$HTTP_CODE" = "201" ] || [ "$HTTP_CODE" = "200" ]; then break; fi
  if [ "$HTTP_CODE" = "429" ] || [ "$HTTP_CODE" = "403" ]; then
    sleep $BACKOFF; BACKOFF=$((BACKOFF * 2)); RETRY_COUNT=$((RETRY_COUNT + 1))
  else
    echo "Failed to post reply (HTTP $HTTP_CODE) — skipping"; break
  fi
done
```

Post-failure: surface the failed comment in the summary so the user can reply manually — a silent drop violates the "every comment replied" invariant.

## Summary report

Group by action class so the user can scan:

1. **Comments fixed + replied** — code changed, committed, reply posted with SHA.
2. **Comments replied without fix** — questions answered, praise acknowledged, clarifications requested, deferrals.
3. **Already replied (skipped)** — idempotency caught these; no action taken.

Under 10 comments, show each inline with its action and reply. 10+, summarize by category with counts and list individual fixed items.

Next steps for the user: review the diff, run tests, push. Skip `"post replies manually"` — they're already posted. For autonomous mode, include the rollback stash name and base SHA, and note explicitly: commits can be reverted, replies cannot.

## Error responses

Phrase errors so the user knows what to do next:

```
PR not found         — Verify the PR number/URL; `gh pr list` to browse.
Not authenticated    — Run: gh auth login
Working tree dirty   — git stash (or commit) before addressing PR comments.
Git user unset       — git config user.name / user.email with instructions.
No comments found    — All reviewers satisfied or comments already addressed.
All already replied  — Nothing to do.
File not found       — Comment references a moved/renamed path; reply asks reviewer to confirm.
Rate limited         — Waiting {N}s before retry ({X}/{MAX}).
Reply post failed    — Comment shown in summary; reply manually.
```

## Usage

```bash
/address-pr-comments 123
/address-pr-comments https://github.com/owner/repo/pull/456
/address-pr-comments 123 --autonomous         # force autonomous
/address-pr-comments 123 --interactive        # force interactive
```

In GitHub Actions, CI env vars trigger autonomous automatically:
```yaml
- run: claude /address-pr-comments ${{ github.event.pull_request.number }}
```
