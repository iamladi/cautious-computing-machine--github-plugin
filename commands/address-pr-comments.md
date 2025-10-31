---
description: Interactive PR comment resolution workflow
---

# Address PR Comments Command

Interactive PR comment resolution workflow for addressing reviewer feedback efficiently.

User provided: `$ARGUMENTS`

## Workflow

### 1. Fetch PR Data

Extract PR number from arguments (could be number like "18" or URL like "https://github.com/owner/repo/pull/123").

```bash
# Get PR details and comments
gh pr view {PR_NUMBER} --json title,body,state,author,headRefName,baseRefName,url,reviews

# Get PR review comments (file/line comments)
gh api repos/{OWNER}/{REPO}/pulls/{PR_NUMBER}/comments

# Get issue comments (general PR comments)
gh api repos/{OWNER}/{REPO}/issues/{PR_NUMBER}/comments

# Checkout PR branch
gh pr checkout {PR_NUMBER}
```

### 2. Analyze Comments

Parse fetched comments and:
- Filter actionable comments (exclude author's own comments, focus on file/line refs)
- Read affected files for context
- Categorize by type:
  - **CODE**: Logic changes, bug fixes, refactoring suggestions
  - **STYLE**: Formatting, naming conventions, code style
  - **DOCS**: Documentation improvements, comment clarity
  - **TEST**: Test coverage, test improvements
  - **QUESTIONS**: Clarifications needed

### 2.5. Score Comments for Autonomous Mode

For each actionable comment, calculate confidence score (0-100) based on multiple factors:

**Scoring Factors:**

1. **Specificity** (0-30 points):
   - Has file path AND line number: +20 points
   - Has concrete code suggestion or example: +10 points
   - Vague or general comment: +0 points

2. **Language Clarity** (0-25 points):
   - Directive language ("Please change", "Must fix", "Should use"): +25 points
   - Suggestion language ("Consider", "Maybe", "Could"): +15 points
   - Question only ("Why did you...?"): +5 points

3. **Category Confidence** (0-25 points):
   - **STYLE** (formatting, naming): +25 points (highly automatable)
   - **DOCS** (documentation, comments): +20 points
   - **CODE** (logic, refactoring): +15 points (needs more care)
   - **TEST** (test coverage): +10 points
   - **QUESTIONS** (clarifications): +5 points

4. **Reviewer Authority** (0-10 points):
   - Maintainer or repository owner: +10 points
   - Core contributor: +7 points
   - Other contributors: +3 points

5. **Discussion Status** (0-10 points):
   - Marked as "CHANGES_REQUESTED" or blocking: +10 points
   - Unresolved discussion: +7 points
   - Already resolved: +0 points (skip)

**Confidence Thresholds:**

- **High (80-100)**: Auto-address in autonomous mode
  - Clear, specific, safe changes with concrete suggestions
  - Example: "Use const instead of let on line 42"

- **Medium (60-79)**: Present to user in interactive mode
  - Reasonable suggestions but need human judgment
  - Example: "Consider extracting this logic into a separate function"

- **Low (0-59)**: Skip or flag for manual review
  - Vague, ambiguous, or complex changes
  - Example: "This approach might have performance issues"

**Scoring Examples:**

```
Comment: "Please use const instead of let for maxRetries on line 42"
- Specificity: 30 (file+line+suggestion)
- Language: 25 (directive)
- Category: 25 (STYLE)
- Authority: 10 (maintainer)
- Status: 10 (CHANGES_REQUESTED)
= TOTAL: 100 → High confidence, auto-address

Comment: "Consider refactoring this method for better readability"
- Specificity: 0 (no location, vague)
- Language: 15 (suggestion)
- Category: 15 (CODE)
- Authority: 3 (contributor)
- Status: 7 (unresolved)
= TOTAL: 40 → Low confidence, skip

Comment: "Add null check before accessing user.email on line 120"
- Specificity: 30 (file+line+concrete)
- Language: 25 (directive)
- Category: 15 (CODE)
- Authority: 10 (maintainer)
- Status: 10 (blocking)
= TOTAL: 90 → High confidence, auto-address
```

### 2.9. Detect Execution Mode

**Auto-detect if running in non-interactive environment:**

The command automatically detects whether to run in interactive or autonomous mode:

```bash
# Check if stdin is a terminal (TTY)
if [ -t 0 ]; then
  MODE="interactive"
else
  MODE="autonomous"
fi

# Check for CI/CD environment variables
if [ -n "$CI" ] || [ -n "$GITHUB_ACTIONS" ] || [ -n "$JENKINS_URL" ] || [ -n "$GITLAB_CI" ]; then
  MODE="autonomous"
fi
```

**User can override detection with arguments:**
- Arguments contain "auto", "--autonomous", or "--auto" → Force autonomous mode
- Arguments contain "interactive" or "--interactive" → Force interactive mode

**Mode Behaviors:**

**Interactive Mode:**
- Use AskUserQuestion tool to present all comments
- User selects which items to address
- Show all comments regardless of confidence score
- Verbose explanations and confirmations

**Autonomous Mode:**
- Filter comments by confidence threshold (≥80)
- Auto-address high-confidence items without prompting
- Skip low/medium confidence items
- Generate detailed report of actions taken
- Create rollback checkpoint before changes

### 3. Present Options (Interactive Mode) OR Auto-Filter (Autonomous Mode)

**IF MODE = "interactive":**

Display actionable comments in organized format:

```
Found {N} comments to address on PR #{NUMBER}: {TITLE}

Actionable Items:
─────────────────────────────────────────────────────────

1. [CODE] {file_path}:{line_number} (Score: 85)
   Reviewer: @{username}
   Comment: "{comment text}"
   Suggested change: {describe what needs to be done}

2. [STYLE] {file_path}:{line_number} (Score: 95)
   Reviewer: @{username}
   Comment: "{comment text}"
   Suggested change: {describe what needs to be done}

3. [DOCS] {file_path}:{line_number} (Score: 50)
   Reviewer: @{username}
   Comment: "{comment text}"
   Suggested change: {describe what needs to be done}

─────────────────────────────────────────────────────────

Which items would you like me to address?
Options: "1,3,4" | "1-5" | "all" | "1"
```

**Use AskUserQuestion tool** to present the selection options to the user.

**IF MODE = "autonomous":**

Filter and display only high-confidence comments (score ≥ 80):

```
🤖 AUTONOMOUS MODE ACTIVE

Analyzing {N} total comments on PR #{NUMBER}: {TITLE}
Confidence threshold: 80/100

High-Confidence Items (will be addressed automatically):
─────────────────────────────────────────────────────────

1. [STYLE] src/utils.ts:42 (Score: 95)
   Reviewer: @maintainer
   Comment: "Please use const instead of let for maxRetries"
   Action: Change variable declaration

2. [DOCS] README.md:15 (Score: 88)
   Reviewer: @maintainer
   Comment: "Add installation instructions for Docker setup"
   Action: Add documentation section

3. [CODE] src/auth.ts:120 (Score: 90)
   Reviewer: @owner
   Comment: "Add null check before accessing user.email"
   Action: Add safety check

Skipped Items (below confidence threshold):
─────────────────────────────────────────────────────────

4. [CODE] src/api.ts:55 (Score: 65)
   Reviewer: @contributor
   Comment: "Consider refactoring this for better performance"
   Reason: Vague suggestion - medium confidence, requires human review

5. [TEST] tests/auth.test.ts (Score: 45)
   Reviewer: @contributor
   Comment: "Why didn't you add tests for the error case?"
   Reason: Question without clear action - low confidence

6. [QUESTIONS] src/models.ts:78 (Score: 35)
   Reviewer: @contributor
   Comment: "This approach might have issues"
   Reason: Ambiguous feedback - low confidence

─────────────────────────────────────────────────────────

Processing 3 high-confidence items automatically...
```

Proceed to address only the high-confidence items (score ≥ 80).

### 3.5. Create Safety Checkpoint (Autonomous Mode Only)

**Before making any changes in autonomous mode**, create a rollback point:

```bash
# Create timestamp for unique identification
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
STASH_NAME="pre-autonomous-pr-${PR_NUMBER}-${TIMESTAMP}"

# Stash any uncommitted changes (including untracked files)
if [ -n "$(git status --porcelain)" ]; then
  git stash push -u -m "$STASH_NAME"
  echo "✓ Uncommitted changes stashed"
fi

# Record the current commit SHA for reference
ROLLBACK_SHA=$(git rev-parse HEAD)

echo "✓ Safety checkpoint created"
echo "  Stash name: $STASH_NAME"
echo "  Base commit: $ROLLBACK_SHA"
echo ""
```

**Rollback mechanism:**
- All changes are made on the current working tree
- User can easily undo with: `git stash apply stash^{/$STASH_NAME}`
- If satisfied, user can drop the stash: `git stash drop stash^{/$STASH_NAME}`

Include rollback instructions in the final report.

### 4. Address Changes

For each selected item:

1. **Show context**: Display the relevant file section and explain the requested change
2. **Read file**: Use Read tool to get current file content
3. **Apply change**: Use Edit tool to make the requested modification
4. **Report**: `✓ Addressed item {N}: {description}`

Example workflow per item:
```
Addressing item 2: [STYLE] src/utils.ts:42
Comment: "Please use const instead of let for immutable variables"

Reading src/utils.ts...
Found line 42: let maxRetries = 3;

Applying change: Converting let to const...
✓ Changed line 42 from 'let' to 'const'

✓ Addressed item 2: Changed variable declaration to const
```

### 5. Summary

After addressing all selected items:

```bash
# Show what changed
git status --short
git diff --stat
```

**Interactive Mode Report:**

```
📊 Summary
  ✅ {N} comments addressed
  📝 {M} files modified

Changes:
  • {file1}: {description}
  • {file2}: {description}

Remaining: {X} comments still need attention

Next steps:
  1. Review changes: git diff
  2. Run tests: {test command if known}
  3. Commit: git add . && git commit -m "fix: address PR review comments"
  4. Push: git push
```

**Autonomous Mode Report:**

```
🤖 AUTONOMOUS EXECUTION COMPLETE

📊 Statistics:
  ✅ {N} high-confidence comments addressed (score ≥ 80)
  ⏭️  {M} comments skipped (below confidence threshold)
  📝 {X} files modified
  ⏱️  Execution time: {duration}

High-Confidence Items Addressed:
──────────────────────────────────────────────────────────

1. ✓ [STYLE] src/utils.ts:42 (Score: 95)
   Comment: "Please use const instead of let for maxRetries"
   Applied: Changed 'let maxRetries = 3' to 'const maxRetries = 3'

2. ✓ [DOCS] README.md:15 (Score: 88)
   Comment: "Add installation instructions for Docker setup"
   Applied: Added Docker setup section with installation steps

3. ✓ [CODE] src/auth.ts:120 (Score: 90)
   Comment: "Add null check before accessing user.email"
   Applied: Added 'if (user && user.email)' check

Skipped Items Requiring Human Review:
──────────────────────────────────────────────────────────

4. ⏭️ [CODE] src/api.ts:55 (Score: 65 - Medium confidence)
   Comment: "Consider refactoring this for better performance"
   Reason: Vague suggestion without concrete implementation details
   Action: Recommend manual review by developer

5. ⏭️ [TEST] tests/auth.test.ts (Score: 45 - Low confidence)
   Comment: "Why didn't you add tests for the error case?"
   Reason: Question format, no clear action specified
   Action: Requires clarification with reviewer

6. ⏭️ [QUESTIONS] src/models.ts:78 (Score: 35 - Low confidence)
   Comment: "This approach might have issues"
   Reason: Ambiguous feedback without specifics
   Action: Needs detailed discussion with team

🔄 Rollback Information:
──────────────────────────────────────────────────────────
  Stash: pre-autonomous-pr-${PR_NUMBER}-${TIMESTAMP}
  Base commit: ${ROLLBACK_SHA}

  To undo all changes:
    git reset --hard ${ROLLBACK_SHA}
    git stash apply stash^{/pre-autonomous-pr-${PR_NUMBER}-${TIMESTAMP}}

  To keep changes and clean up:
    git stash drop stash^{/pre-autonomous-pr-${PR_NUMBER}-${TIMESTAMP}}

📋 Next Steps:
──────────────────────────────────────────────────────────
  1. Review autonomous changes: git diff
  2. Run tests to verify correctness: {test_command}
  3. Review skipped items manually (shown above)
  4. If satisfied, commit:
     git add . && git commit -m "fix: address PR comments (autonomous: ${N} items)"
  5. Push changes: git push
  6. If issues found, rollback using instructions above
```

## Rules

### Prioritization

**High Priority** (address first):
- File/line-specific comments with concrete suggestions
- Comments using directive language ("Please change...", "Must fix...")
- Maintainer/owner feedback
- Unresolved discussions marked as blocking

**Lower Priority**:
- General discussion comments
- Questions without suggested changes
- Already resolved threads
- Praise/approval comments

### Skip These

Don't process:
- Info-only comments without action items
- Questions that just need answers (not code changes)
- Comments marked as resolved or outdated
- Praise/acknowledgment comments
- Comments from the PR author themselves

### Safety

- **No automatic git operations**: Never auto-commit or push
- **Show before doing**: Display planned changes before applying
- **Preserve context**: Don't remove important code/comments
- **Ask when unclear**: Use AskUserQuestion if comment is ambiguous

## Usage

**Interactive Mode (default when run in terminal):**

```bash
/address-pr-comments 18                                    # PR number
/address-pr-comments https://github.com/owner/repo/pull/123   # PR URL
```

**Autonomous Mode (auto-detected in CI/CD environments):**

The command automatically switches to autonomous mode when:
- Running in CI/CD environment (CI=true, GITHUB_ACTIONS=true, JENKINS_URL, GITLAB_CI, etc.)
- stdin is not a TTY (piped or redirected input)

```bash
# Runs autonomously if detected in CI environment
/address-pr-comments 18

# In CI/CD workflow:
- name: Address PR Comments
  run: claude /address-pr-comments ${{ github.event.pull_request.number }}
  # Automatically runs in autonomous mode
```

**Force Specific Mode:**

```bash
# Force autonomous mode (even in interactive terminal)
/address-pr-comments 18 --autonomous
/address-pr-comments 18 --auto
/address-pr-comments 18 auto

# Force interactive mode (even in CI/CD)
/address-pr-comments 18 --interactive
/address-pr-comments 18 interactive
```

**Environment Variables for Mode Detection:**

The command checks these environment variables to detect CI/CD:
- `CI=true` (generic CI indicator)
- `GITHUB_ACTIONS=true` (GitHub Actions)
- `JENKINS_URL` (Jenkins)
- `GITLAB_CI` (GitLab CI)
- `CIRCLECI=true` (CircleCI)
- `TRAVIS=true` (Travis CI)

## Error Handling

### PR doesn't exist
```
❌ Error: PR #{NUMBER} not found
   Verify the PR number/URL and try again.
   Tip: Use `gh pr list` to see available PRs
```

### No actionable comments
```
✅ No actionable comments found on PR #{NUMBER}
   All comments are either:
   - Already addressed
   - Questions/discussions without code change requests
   - From the PR author
```

### Not authenticated with GitHub
```
❌ Error: Not authenticated with GitHub CLI
   Run: gh auth login
```

### Uncommitted changes
```
⚠️  WARNING: You have uncommitted changes
    Stash them first: git stash
    Or commit them before addressing PR comments

    Continue anyway? (not recommended)
```

### File not found
```
⚠️  Comment references {file_path} which doesn't exist
    This comment may be outdated or refers to a renamed file.
    Skipping...
```

### Ambiguous comment
```
❓ Comment unclear: "{comment text}"
   What change would you like me to make?

   [Use AskUserQuestion to clarify with user]
```

## Implementation Notes

### General
- Use `gh api` for detailed comment data with file paths and line numbers
- Parse JSON responses to extract actionable items
- Use Read tool to get file context before changes
- Use Edit tool for precise line modifications
- Track which comments are addressed for final report
- Consider comment timestamps to prioritize recent feedback

### Confidence Scoring Algorithm

**Step 1: Parse comment structure**
```javascript
{
  path: "src/utils.ts",           // File path (if available)
  line: 42,                        // Line number (if available)
  body: "Please use const...",     // Comment text
  user: { login: "reviewer" },    // Commenter info
  author_association: "OWNER",     // Reviewer role
  state: "unresolved"              // Discussion status
}
```

**Step 2: Calculate scores**
- Specificity: Check for `path` AND `line` presence, analyze `body` for code examples
- Language: Regex patterns for directive words ("please", "must", "should" vs "consider", "maybe")
- Category: NLP-based classification or keyword matching
- Authority: Map `author_association` (OWNER→10, COLLABORATOR→7, CONTRIBUTOR→3)
- Status: Check review state and `state` field

**Step 3: Apply thresholds**
```
if (score >= 80) {
  autonomous_mode: address_automatically
} else if (score >= 60) {
  interactive_mode: present_to_user
} else {
  skip_or_flag_for_manual_review
}
```

### Mode Detection Implementation

```bash
# Pseudo-code for mode detection
function detect_mode() {
  # Check explicit override first
  if args contains "--autonomous" or "--auto" or "auto":
    return "autonomous"
  if args contains "--interactive" or "interactive":
    return "interactive"

  # Check CI environment variables
  if $CI or $GITHUB_ACTIONS or $JENKINS_URL or $GITLAB_CI or $CIRCLECI or $TRAVIS:
    return "autonomous"

  # Check if stdin is a terminal
  if not is_tty(stdin):
    return "autonomous"

  # Default to interactive
  return "interactive"
}
```

### Safety Mechanisms

**Autonomous mode only:**
1. Create git stash before any modifications
2. Filter to high-confidence items only (score ≥ 80)
3. Track all changes for detailed reporting
4. Provide clear rollback instructions
5. Never auto-commit or auto-push

**Both modes:**
1. Read file context before applying changes
2. Show diffs for user review
3. Skip ambiguous or unclear comments
4. Preserve code context and intent
