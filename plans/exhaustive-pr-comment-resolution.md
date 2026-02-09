---
title: "Exhaustive PR Comment Resolution with Mandatory Replies"
type: Enhancement
issue: null
research: ["research/research-address-pr-comments-improvement.md"]
status: Ready for Implementation
reviewed: true
reviewers: ["codex", "gemini"]
created: 2026-02-09
---

# PRD: Exhaustive PR Comment Resolution with Mandatory Replies

## Metadata
- **Type**: Enhancement
- **Priority**: High
- **Severity**: N/A
- **Estimated Complexity**: 5
- **Created**: 2026-02-09
- **Status**: Ready for Implementation

## Overview

### Problem Statement

The `/github:address-pr-comments` command currently has three critical gaps:

1. **No GitHub replies posted.** The command applies code fixes locally but never posts sub-comments on the PR. The reviewer has no way to know which comments were addressed or why something was skipped — they must manually diff the code and guess.

2. **Silent comment skipping.** Entire categories of comments (questions, praise, resolved threads, low-confidence items) are filtered out before processing. No trace is left on GitHub explaining what happened to these comments.

3. **No auto-commit.** Fixes are applied to the working tree but never committed, so there's no SHA to reference in replies and no atomic traceability between a review comment and its fix.

The result: after running `/address-pr-comments`, the reviewer still has to manually walk through every comment to verify what was addressed. This defeats the purpose of the command.

### Goals & Objectives

1. Every comment fetched from the PR must receive a sub-comment reply explaining the action taken (fix, explanation, acknowledgment) or why no action was taken
2. Code fixes must be committed immediately so replies can reference specific commit SHAs
3. All three comment types must be fetched (review comments, review summaries, issue comments) — no blind spots
4. The command must produce a clear audit trail: for any PR comment, you can see the reply explaining what happened

### Success Metrics

- **Primary Metric**: 100% reply coverage — every non-author, non-bot, top-level comment on the PR receives a sub-comment reply after the command runs
- **Secondary Metrics**:
  - Every code fix has a dedicated commit with a descriptive message
  - Every "Fixed" reply references a specific commit SHA
  - Zero comments silently filtered without trace
- **Quality Gates**:
  - Manual test: run on a PR with mixed comment types (code fix, question, praise, unclear suggestion), verify all get replies

## User Stories

### Story 1: Know what happened to every comment
- **As a**: PR author using `/address-pr-comments`
- **I want**: every reviewer comment to receive a reply explaining the action taken
- **So that**: I can see at a glance which comments are fixed, which need discussion, and which are acknowledged — without manually comparing code diffs to comment threads
- **Acceptance Criteria**:
  - [ ] Every top-level reviewer comment has a sub-comment reply after the command runs
  - [ ] Fixes reference specific commit SHAs
  - [ ] Explanations provide meaningful context, not generic acknowledgments
  - [ ] Questions get thoughtful answers based on code context

### Story 2: Trace fixes to comments
- **As a**: reviewer checking if my feedback was addressed
- **I want**: each code fix to be in its own commit, referenced in the reply to my comment
- **So that**: I can click the SHA in the reply and see exactly what changed for my specific comment
- **Acceptance Criteria**:
  - [ ] Each fix is its own commit (not batched)
  - [ ] Commit message references the reviewer and comment
  - [ ] Reply contains "Fixed in {sha}" linking to the commit

### Story 3: Understand skipped items
- **As a**: PR author running the command autonomously
- **I want**: below-threshold or unclear comments to still get a reply explaining why they weren't auto-fixed
- **So that**: I know the command saw them and can decide whether to handle them manually
- **Acceptance Criteria**:
  - [ ] Low-confidence comments get a reply explaining they need manual review
  - [ ] Unclear comments get a reply asking for clarification
  - [ ] Praise/LGTM comments get an acknowledgment reply

## Requirements

### Functional Requirements

1. **FR-1**: Fetch all three comment types from GitHub API with pagination
   - Details: Review comments (`/pulls/{pr}/comments`), review summaries (`/pulls/{pr}/reviews`), and issue comments (`/issues/{pr}/comments`). All fetches MUST use `gh api --paginate` to handle PRs with >30 comments (GitHub's default page size).
   - Priority: Must Have

2. **FR-2**: Process every non-author, non-bot, top-level comment — no silent filtering
   - Details: PR author's own comments and bot comments are excluded (author = PR author, not command invoker — if a maintainer runs the command, the PR author's comments are still excluded). Replies (`in_reply_to_id != null`) are excluded. Everything else MUST be processed and replied to.
   - Priority: Must Have

3. **FR-3**: Post a sub-comment reply for every processed comment
   - Details: Review comments use `/pulls/{pr}/comments/{id}/replies` for threaded replies. Issue comments and review summaries use a new top-level PR comment with `> quoted original` text for clear linking.
   - Priority: Must Have

4. **FR-4**: Auto-commit each code fix individually
   - Details: After applying a fix via Edit tool, immediately `git add {file} && git commit -m "fix: {description}"`. Capture SHA via `git rev-parse --short HEAD` for use in the reply.
   - Priority: Must Have

5. **FR-5**: Categorize every comment and select the appropriate reply template
   - Details: Categories: `actionable/clear` (fix + "Fixed in {sha}"), `actionable/unclear` (reply asking clarification), `not-actionable/question` (reply with explanation), `not-actionable/praise` (reply with acknowledgment), `not-actionable/out-of-scope` (reply with deferral), `below-threshold` (reply explaining needs manual review in autonomous mode).
   - Priority: Must Have

6. **FR-6**: Maintain confidence scoring for autonomous mode
   - Details: Keep the existing 0-100 scoring system. In autonomous mode, high-confidence items get auto-fixed. Below-threshold items still get a REPLY (not silently skipped) explaining they need manual review.
   - Priority: Must Have

7. **FR-7**: Interactive mode must process ALL comments
   - Details: In interactive mode, user selects which comments to fix. Comments the user does NOT select still get a reply: "Noted — deferring to manual review" or similar. No comment is left without a reply.
   - Priority: Must Have

8. **FR-8**: Idempotent re-runs — skip comments that already have a reply from the command user
   - Details: Before posting a reply, check if a reply from the current `gh` user already exists on that comment (by fetching existing replies and checking `user.login`). If a reply already exists, skip posting and note "Already replied" in the report. This prevents duplicate replies on re-runs.
   - Priority: Must Have

9. **FR-9**: Safe reply body construction — no shell injection
   - Details: Reply bodies MUST NOT be passed via shell string interpolation. Use `gh api --input -` with JSON piped from `jq` to safely handle code blocks, quotes, backticks, `$variables`, and multiline content in comment bodies.
   - Priority: Must Have

10. **FR-10**: Line-drift-safe code location for multi-comment files
    - Details: When multiple comments reference the same file, earlier fixes may shift line numbers. The command MUST NOT rely solely on the API's `line` field. Instead, use the `diff_hunk` from the comment payload to identify the unique code context (anchor text) and locate it dynamically in the current file state before applying each fix.
    - Priority: Must Have

### Non-Functional Requirements

1. **NFR-1**: GitHub API rate limiting with retry/backoff
   - Requirement: Handle rate limits gracefully when posting many replies. On 403/429 responses, read `Retry-After` or `X-RateLimit-Reset` headers and wait accordingly. Use exponential backoff (1s, 2s, 4s) with max 3 retries per request.
   - Target: Support PRs with up to 50 comments without unrecoverable rate limit errors
   - Measurement: No permanent 403/429 failures; transient ones retried successfully

2. **NFR-2**: Reply quality
   - Requirement: Replies must be contextual and useful, not generic boilerplate
   - Target: Fixed replies reference SHAs, explanations reference actual code, acknowledgments are brief
   - Measurement: Manual review of reply quality

3. **NFR-3**: Preflight validation
   - Requirement: Before processing, verify: (a) `gh auth status` succeeds with write permissions, (b) working tree is clean (`git status --porcelain` is empty — abort with warning if dirty), (c) git identity is configured (`git config user.name` and `user.email` are set)
   - Target: Fail fast with clear error messages before any commits or replies are posted
   - Measurement: Preflight catches all known setup issues

### Technical Requirements

- **Stack**: Markdown command file (no TypeScript needed — this is a prompt-based command)
- **Dependencies**: `gh` CLI (already required), git (already required)
- **Architecture**: Single command file `address-pr-comments.md` rewrite. No new agents, skills, or plugins.
- **API Contracts**: Uses existing GitHub REST API endpoints via `gh api`

## Scope

### In Scope

- Rewrite `github-plugin/commands/address-pr-comments.md` to implement exhaustive comment resolution with mandatory replies
- Add review summary fetching (3rd comment type)
- Add per-fix auto-commit behavior
- Add reply posting via `gh api` for all comment types
- Add reply templates for all categories (fix, explanation, clarification, acknowledgment, deferral, manual-review-needed)
- Add quote-based reply format for issue comments (no threading API available)
- Retain interactive/autonomous mode detection
- Retain confidence scoring system
- Update summary/report format to reflect new reply-posting behavior

### Out of Scope

- Workflows plugin changes (separate concern, different architecture)
- Swarm/team-based parallel processing (that's `/workflows:resolve-comments --swarm`)
- Iterative loop for new comments during processing (that's the workflows-plugin's responsibility)
- Auto-push (user must still push manually)
- GitHub GraphQL API (stick with REST via `gh api`)

### Future Considerations

- Integration between `/github:address-pr-comments` and `/workflows:resolve-comments` to avoid duplication
- Auto-push option for CI/CD environments
- Support for GitHub's "resolve conversation" API if it becomes available via REST

## Impact Analysis

### Affected Areas

- `github-plugin/commands/address-pr-comments.md` — complete rewrite
- GitHub PR comment threads — the command will now post replies (visible to reviewers)

### Users Affected

- PR authors who use `/address-pr-comments` — now see replies posted automatically
- PR reviewers — now receive sub-comment replies for each of their comments

### System Impact

- **Performance**: More GitHub API calls (one POST per comment for replies). Mitigated by sequential processing with brief pauses if needed.
- **Security**: No new credentials needed — uses existing `gh` auth.
- **Data Integrity**: Commits are created automatically. Each commit is atomic (one fix per commit). No destructive operations.

### Dependencies

- **Upstream**: GitHub API availability, `gh` CLI authentication
- **Downstream**: None — this is a leaf command
- **External**: GitHub REST API (`/pulls/{pr}/comments/{id}/replies`, `/issues/{pr}/comments`)

### Breaking Changes

- [x] **Behavioral change**: Command now auto-commits fixes (previously: no commits). Users who relied on "apply edits without committing" behavior will see commits created.
- [x] **Behavioral change**: Command now posts GitHub replies (previously: local-only output). This is visible to reviewers.
- [x] **"Skip These" section removed**: Questions, praise, and info-only comments are no longer skipped — they all get replies.

## Solution Design

### Approach

Rewrite `address-pr-comments.md` as a single comprehensive command that processes every comment in a linear pass:

1. **Fetch** all three comment types (review comments, review summaries, issue comments)
2. **Filter** only author's own comments, bot comments, and reply comments (`in_reply_to_id != null`)
3. **For each remaining comment**, categorize and act:
   - `actionable/clear` → Read file → Edit → Commit → Reply "Fixed in {sha}"
   - `actionable/unclear` → Reply asking for clarification
   - `not-actionable/question` → Read relevant code → Reply with explanation
   - `not-actionable/praise` → Reply "Acknowledged, thank you!"
   - `not-actionable/out-of-scope` → Reply "Out of scope for this PR, will address in follow-up"
   - `below-threshold` (autonomous only) → Reply "This comment needs manual review: {reason}"
4. **Reply posting** uses the correct API endpoint per comment type:
   - Review comments: `gh api -X POST repos/{o}/{r}/pulls/{pr}/comments/{id}/replies -f body="..."`
   - Issue comments: `gh api -X POST repos/{o}/{r}/issues/{pr}/comments -f body="> {quoted original}\n\n{reply}"`
   - Review summaries: `gh api -X POST repos/{o}/{r}/issues/{pr}/comments -f body="> From @{reviewer}'s review:\n> {quoted body}\n\n{reply}"`
5. **Summary** includes all comments with their actions and reply status

### Alternatives Considered

1. **Delegate to workflows-plugin's comment-resolver agent**
   - Pros: Reuse existing categorization and reply logic
   - Cons: Creates cross-plugin dependency; workflows-plugin has its own skip behavior that would need fixing; different architectural model (skill + agent vs single command)
   - Why rejected: The github-plugin should remain standalone. The user chose github-plugin-only scope.

2. **Add reply-posting as optional flag (--reply)**
   - Pros: Backwards compatible, opt-in
   - Cons: Defeats the purpose — the whole point is that EVERY run posts replies. Making it optional means the default behavior remains broken.
   - Why rejected: The core problem IS the default behavior. Making replies mandatory is the fix.

3. **Post replies in batch after all fixes applied**
   - Pros: Fewer API round-trips
   - Cons: If the command crashes mid-way, no replies posted. Per-comment posting is more resilient.
   - Why rejected: Per-comment is more reliable and gives immediate feedback.

### Data Model Changes

None — this is a prompt-based command, no data models.

### API Changes

No new APIs created. Uses existing GitHub REST API endpoints:
- `GET /repos/{o}/{r}/pulls/{pr}/comments` (already used)
- `GET /repos/{o}/{r}/issues/{pr}/comments` (already used)
- `GET /repos/{o}/{r}/pulls/{pr}/reviews` (NEW — was missing)
- `POST /repos/{o}/{r}/pulls/{pr}/comments/{id}/replies` (NEW — for review comment replies)
- `POST /repos/{o}/{r}/issues/{pr}/comments` (NEW — for issue comment and review summary replies)

### UI/UX Changes

Terminal output changes:
- Each processed comment now shows: category, action taken, reply posted status
- Summary includes reply count alongside fix count
- No more "Skipped Items" section without replies — all items show their reply

## Implementation Plan

### Phase 1: Preflight Checks and Comment Fetching
**Complexity**: 3 | **Priority**: High

<!-- Addressed: preflight validation (Codex: auth/permission, Gemini: git identity), pagination (Codex), dirty working tree (Codex) -->

- [ ] Add preflight checks before any processing:
  - Verify `gh auth status` succeeds (abort with "Run: gh auth login" if not)
  - Verify working tree is clean via `git status --porcelain` (abort with "Stash or commit changes first" if dirty — do not proceed with uncommitted changes to avoid accidental staging)
  - Verify `git config user.name` and `git config user.email` are set (abort with config instructions if missing)
- [ ] Add review summary fetching (`gh api --paginate repos/{o}/{r}/pulls/{pr}/reviews`)
- [ ] Add `--paginate` flag to ALL three `gh api` fetch calls to handle PRs with >30 comments
- [ ] Unify all three comment types into a single list with normalized structure (`id`, `type`, `path`, `line`, `diff_hunk`, `body`, `user`, `created_at`, `in_reply_to_id`). Include `diff_hunk` for review comments — it's needed for line-drift-safe code location.
- [ ] Update filtering: only exclude PR author's own comments, bot comments, and replies (`in_reply_to_id != null`). Remove the "Skip These" section that filters out questions, praise, resolved, and info-only comments. Filter out comments with empty/whitespace-only bodies (review summaries can have null body).
- [ ] Keep confidence scoring system intact (still useful for autonomous mode prioritization)
- [ ] Add idempotency check: for each comment, fetch existing replies and check if one from the current `gh` user already exists. Mark these as "already replied" and skip them in processing.

### Phase 2: Add Per-Comment Processing with Mandatory Replies
**Complexity**: 5 | **Priority**: High

<!-- Addressed: line drift (Gemini), multi-file/no-change commits (Codex), outdated file handling (Codex) -->

- [ ] Define the complete category set: `actionable/clear`, `actionable/unclear`, `not-actionable/question`, `not-actionable/praise`, `not-actionable/out-of-scope`, `below-threshold`
- [ ] For `actionable/clear`: implement line-drift-safe fix application:
  1. Read the file via Read tool
  2. Locate the code using `diff_hunk` context (anchor text matching), NOT the API `line` number alone — this handles line drift from earlier fixes on the same file
  3. Apply fix via Edit tool
  4. Check if file actually changed: `git diff --quiet {file}`. If no change (edit was no-op), skip commit and reply "No change needed — code already matches the suggestion"
  5. Stage ONLY the specific file(s) touched: `git add {file}` (never `git add .` or `git add -A`)
  6. Commit: `git commit -m "fix: {desc}\n\nAddresses comment from @{reviewer}\nComment ID: {id}"`
  7. Capture SHA: `git rev-parse --short HEAD`
  8. Reply "Fixed in {sha}"
- [ ] Handle missing/renamed files: if the file referenced in the comment doesn't exist, reply "This file appears to have been moved or renamed since this review. Could you check if this comment is still relevant?" — do not attempt to edit.
- [ ] For `actionable/unclear`: Reply "Could you clarify what specific change you're looking for? For example:\n- Should I {option A}?\n- Or {option B}?"
- [ ] For `not-actionable/question`: Read relevant code context → Reply with explanation of why the code is written that way, offer to adjust
- [ ] For `not-actionable/praise`: Reply "Acknowledged, thank you!"
- [ ] For `not-actionable/out-of-scope`: Reply "Out of scope for this PR — will address in follow-up"
- [ ] For `below-threshold` (autonomous mode only): Reply "This comment requires manual review. Reason: {scoring-reason}. The automated system wasn't confident enough to apply a fix autonomously."
- [ ] In interactive mode: user-selected comments get fixed + replied. Unselected comments get reply "Noted — deferring to manual review"

### Phase 3: Add Reply Posting Logic
**Complexity**: 3 | **Priority**: High

<!-- Addressed: shell injection (Consensus), rate limit retry/backoff (Codex), review summary empty body (Codex) -->

- [ ] **CRITICAL: Safe body construction.** All reply bodies MUST be passed via `gh api --input -`, NOT via `-f body="..."` shell interpolation. Construct JSON payload using `jq`: `jq -n --arg b "$REPLY_BODY" '{body:$b}' | gh api -X POST "..." --input -`. This prevents shell injection from code blocks, backticks, `$variables`, and special chars in comment bodies.
- [ ] For review comments (type: "review"): `jq -n --arg b "$REPLY" '{body:$b}' | gh api -X POST "repos/{o}/{r}/pulls/{pr}/comments/{id}/replies" --input -`
- [ ] For issue comments (type: "issue"): Construct body as `> {quoted_original_body_truncated_200_chars}\n\n{reply}`, then pipe via jq as above
- [ ] For review summaries (type: "review-summary"): Construct body as `> From @{reviewer}'s review ({state}):\n> {quoted_body_truncated_200_chars}\n\n{reply}`. Handle empty/null review bodies: if body is empty, use "No summary text provided — addressing based on review state ({state})."
- [ ] Add error handling with retry: if reply POST returns 403/429, read `Retry-After` header and wait (exponential backoff: 1s, 2s, 4s, max 3 retries). On other errors, log and continue processing.
- [ ] Skip posting if idempotency check (Phase 1) found existing reply from current user

### Phase 4: Update Summary and Report Format
**Complexity**: 2 | **Priority**: Medium

- [ ] Update interactive mode summary to show reply status for each comment
- [ ] Update autonomous mode summary to show: comments fixed + replied, comments replied-only, total replies posted
- [ ] Remove "Skipped Items Requiring Human Review" framing — replace with "Comments Replied Without Fix" showing the reply that was posted
- [ ] Update "Next Steps" to remove manual reply instructions (replies already posted) — keep manual push instruction
- [ ] Keep rollback section for autonomous mode (rollback only reverts commits, replies remain on GitHub — note this in rollback section)

### Phase 5: Validation
**Complexity**: 2 | **Priority**: High

- [ ] Plugin validation: `cd github-plugin && bun run validate`
- [ ] Manual test: run on a real PR with mixed comment types
- [ ] Verify all comments received replies on GitHub
- [ ] Verify each fix has its own commit with descriptive message
- [ ] Verify autonomous mode posts replies for below-threshold items instead of silently skipping
- [ ] Verify interactive mode posts replies for unselected items

## Relevant Files

### Existing Files

- `github-plugin/commands/address-pr-comments.md` — THE file being rewritten. Contains the complete current command definition (604 lines).
- `github-plugin/.claude-plugin/plugin.json` — Plugin manifest. May need description update if command behavior changes significantly.
- `github-plugin/README.md` — Plugin documentation. Needs update to reflect new reply-posting and auto-commit behavior.

### New Files

None — this is a rewrite of an existing file.

### Test Files

None — this is a markdown command file, not code. Validation is manual testing against real PRs.

## Testing Strategy

### Unit Tests

N/A — markdown command files don't have unit tests.

### Integration Tests

N/A — testing requires a real GitHub PR with comments.

### E2E Tests

N/A — would require a real PR environment.

### Manual Test Cases

1. **Test Case: Mixed comment types**
   - Steps:
     1. Create a PR with: 1 clear code suggestion, 1 vague suggestion, 1 question, 1 "LGTM" praise, 1 general discussion
     2. Run `/address-pr-comments {PR_NUMBER}`
     3. Check GitHub PR page
   - Expected: All 5 comments have sub-comment replies. The code suggestion has a fix commit + "Fixed in {sha}" reply.

2. **Test Case: Autonomous mode below-threshold**
   - Steps:
     1. Run `/address-pr-comments {PR_NUMBER} --auto` on a PR with low-confidence comments
     2. Check GitHub PR page
   - Expected: Low-confidence comments have replies explaining they need manual review (not silently skipped).

3. **Test Case: Issue comment reply format**
   - Steps:
     1. Create a PR with a general (non-file-specific) comment
     2. Run `/address-pr-comments {PR_NUMBER}`
     3. Check GitHub PR page
   - Expected: A new comment appears with `> quoted original` text and the reply below.

4. **Test Case: Review summary**
   - Steps:
     1. Submit a review with "Request Changes" and a summary comment
     2. Run `/address-pr-comments {PR_NUMBER}`
     3. Check GitHub PR page
   - Expected: A new PR comment appears addressing the review summary with quote.

5. **Test Case: Idempotent re-run**
   - Steps:
     1. Run `/address-pr-comments {PR_NUMBER}` on a PR (first run)
     2. Run `/address-pr-comments {PR_NUMBER}` again (second run)
     3. Check GitHub PR page
   - Expected: No duplicate replies. Second run reports "Already replied" for all comments.

6. **Test Case: Multiple comments on same file (line drift)**
   - Steps:
     1. Create a PR where 2+ review comments reference different lines of the same file
     2. Run `/address-pr-comments {PR_NUMBER}`
   - Expected: Both fixes are applied correctly even though the first fix may have shifted line numbers.

7. **Test Case: Comment body with special characters**
   - Steps:
     1. Create a PR comment containing: code blocks with backticks, `$variables`, double quotes, newlines
     2. Run `/address-pr-comments {PR_NUMBER}`
   - Expected: Reply is posted successfully with properly quoted original text. No shell errors.

8. **Test Case: Dirty working tree**
   - Steps:
     1. Make uncommitted changes in the repo
     2. Run `/address-pr-comments {PR_NUMBER}`
   - Expected: Command aborts with clear message to stash or commit first.

## Risk Assessment

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| GitHub API rate limiting with many replies | Medium | Medium | Exponential backoff with retry on 403/429. Read `Retry-After` header. Max 3 retries per request. |
| Reply POST fails mid-processing | Low | Medium | Continue processing remaining comments. Log failed replies in summary. Retry with backoff first. |
| Auto-commit creates messy git history | Low | Low | Each commit has descriptive message with reviewer attribution. User can squash before push if desired. |
| Review summary quotes are too long | Low | Low | Truncate quoted body to first 200 chars with "..." if longer. |
| Line number drift on same-file edits | Medium | High | Use `diff_hunk` anchor text for code location, not API `line` number alone. Re-read file before each fix. |
| Shell injection from comment content | Medium | Critical | All reply bodies piped via `jq` + `--input -`. No shell string interpolation of user content. |
| Duplicate replies on re-run | Medium | Medium | Idempotency check: skip comments that already have a reply from the current user. |
| Dirty working tree causes accidental staging | Low | High | Preflight check: abort if `git status --porcelain` is non-empty. |

### Business Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Reviewers annoyed by automated replies | Low | Medium | Replies are contextual and useful, not spam. Each reply explains what was done. |
| Posting wrong reply to wrong comment | Very Low | High | Each reply is posted immediately after processing its specific comment — no batching confusion. |

### Mitigation Strategy

Sequential processing (one comment at a time) with immediate reply posting ensures correctness. If any step fails, the error is logged and processing continues for remaining comments. The summary report at the end shows which replies succeeded and which failed.

## Rollback Strategy

### Rollback Steps

1. **Git rollback**: `git reset --soft HEAD~{N}` to undo the N fix commits (preserves changes in working tree)
2. **Full revert**: `git reset --hard {ROLLBACK_SHA}` to revert to pre-command state
3. **GitHub replies cannot be automatically deleted** — note this clearly in the command output. Replies posted to GitHub are permanent unless manually deleted.

### Rollback Conditions

- Incorrect fixes were applied (wrong code changes)
- Replies were posted to wrong comments
- Rate limit errors caused partial processing

## Validation Commands

```bash
# Validate plugin manifest
cd /Users/iamladi/Projects/claude-code-plugins/github-plugin && bun run validate

# Verify the command file exists and has expected structure
wc -l /Users/iamladi/Projects/claude-code-plugins/github-plugin/commands/address-pr-comments.md

# Manual validation: run on a test PR
# /address-pr-comments {test-pr-number}
# Then check GitHub PR page for replies
```

## Acceptance Criteria

- [ ] All three comment types fetched (review comments, review summaries, issue comments)
- [ ] Every non-author, non-bot, top-level comment receives a sub-comment reply
- [ ] Code fixes auto-committed with descriptive messages referencing reviewer and comment ID
- [ ] "Fixed in {sha}" replies reference actual commit SHAs
- [ ] Review comment replies are threaded (appear under the original comment)
- [ ] Issue comment replies use quote format (`> original\n\nreply`)
- [ ] Review summary replies use quote format with reviewer attribution
- [ ] Autonomous mode: below-threshold comments get replies (not silently skipped)
- [ ] Interactive mode: unselected comments get "deferring to manual review" replies
- [ ] Praise/LGTM comments get acknowledgment replies
- [ ] Questions get contextual explanation replies
- [ ] Summary report shows all comments with their actions and reply status
- [ ] Plugin validation passes (`bun run validate`)
- [ ] No silent filtering — the "Skip These" section is removed

## Dependencies

### New Dependencies

None — uses existing `gh` CLI and git.

### Dependency Updates

None.

## Notes & Context

### Additional Context

- The workflows-plugin has a parallel implementation (`/workflows:resolve-comments`) that is more sophisticated (iterative loop, swarm mode, dedicated agent). This plan intentionally does NOT modify the workflows-plugin — it focuses on making the github-plugin's command self-sufficient.
- The workflows-plugin's `comment-resolver` agent has useful reply templates (lines 121-157 of `workflows-plugin/agents/comment-resolver.md`) that inform the template design in this plan, but the implementation is independent.

### Assumptions

- `gh` CLI is authenticated and has write access to post comments
- The PR exists and is not merged
- The user is running from within the git repository
- GitHub API reply endpoints work as documented (POST to `/pulls/{pr}/comments/{id}/replies` creates threaded reply)

### Constraints

- GitHub REST API does not support threaded replies on issue comments — must use quote format
- GitHub REST API does not have a direct reply endpoint for review summaries — must post as new issue comment
- Reply text passed via `gh api -f body="..."` must handle special characters in quoted text

### Related Tasks/Issues

- Research document: `research/research-address-pr-comments-improvement.md`
- Workflows plugin comment resolution: `workflows-plugin/skills/comment-resolution/SKILL.md`

### References

- GitHub REST API: Pull Request Review Comments — https://docs.github.com/en/rest/pulls/comments
- GitHub REST API: Issue Comments — https://docs.github.com/en/rest/issues/comments
- GitHub REST API: Pull Request Reviews — https://docs.github.com/en/rest/pulls/reviews

### Open Questions

- [x] Commit strategy? → Auto-commit per fix (user decision)
- [x] Praise handling? → Reply to everything (user decision)
- [x] Issue comment threading? → Quote + new comment (user decision)
- [x] Scope? → GitHub plugin only (user decision)

## Blindspot Review

**Reviewers**: GPT-5.2-Codex (xhigh), Gemini 3 Pro
**Date**: 2026-02-09
**Plan Readiness**: Ready

### Addressed Concerns

- [Consensus, Critical] Shell injection / text escaping in reply bodies → Added FR-9 (safe body construction via jq + --input -), updated Phase 3 with explicit safe payload method
- [Gemini, Critical] Line number drift when multiple comments on same file → Added FR-10 (diff_hunk anchor text matching), updated Phase 2 with line-drift-safe fix application
- [Codex, High] Pagination missing on API fetches → Updated FR-1 to require `--paginate` on all fetches, updated Phase 1
- [Codex, High] No idempotency / duplicate reply handling → Added FR-8 (check existing replies before posting), updated Phase 1 and Phase 3
- [Codex, High] Dirty working tree risk → Added to NFR-3 preflight checks (abort if dirty), updated Phase 1
- [Codex, Medium] Multi-file and no-change commit gaps → Updated Phase 2 to check `git diff --quiet` before committing, stage only specific files
- [Codex, Medium] Outdated file / missing file handling → Added to Phase 2 (reply with "file appears moved/renamed")
- [Codex, Medium] Rate limit retry/backoff underspecified → Updated NFR-1 with exponential backoff and Retry-After header handling
- [Codex, Medium] Auth and permission failure path missing → Added to NFR-3 preflight checks (`gh auth status`)
- [Gemini, Medium] Git identity configuration → Added to NFR-3 preflight checks (`git config user.name/email`)
- [Codex, Low] Author definition ambiguity → Clarified in FR-2 (author = PR author, not command invoker)
- [Codex, Low] Review summary empty body edge case → Updated Phase 3 to handle empty/null review bodies
- [Codex, Medium] Testing gaps for idempotency and special chars → Added manual test cases 5-8

### Acknowledged but Deferred

- [Gemini, Medium] Broken intermediate commits — individual commits may not pass CI if a rename spans multiple files referenced in separate comments. Deferred because: (a) this is inherent to per-fix commits and the user explicitly chose this strategy, (b) user can squash before push, (c) the alternative (batching) breaks the traceability that is the primary goal.

### Dismissed

- [Gemini, Low, 0.7] Pending/draft review comments — `gh api` for review comments (`/pulls/{pr}/comments`) only returns comments from submitted reviews by default. Pending/draft review comments are not visible via the REST API to other users. No action needed.
