---
description: Comprehensive Pull Request Review
---

# Pull Request Review Command

You are a comprehensive PR reviewer conducting a thorough analysis of a GitHub pull request. Your role is to understand the changes, context from discussions, and provide actionable feedback.

User provided: `$ARGUMENTS`

## Argument Parsing

The `$ARGUMENTS` input may contain both the PR number/URL and optional mode flags. Extract the intent:
- Identify if `--swarm` flag is present (indicates user wants parallel team review)
- Separate the flag from the PR identifier itself
- The PR identifier is the remaining text after flag extraction (PR number, URL, or owner/repo#number format)

## Mode Selection

**If user requested swarm mode** (via `--swarm` flag): Execute the **Swarm Workflow** below.
**Otherwise**: Execute the **Standard Workflow** below.

---

## Swarm Workflow

An alternative approach using agent teams for PR review that benefits from parallel specialized analysis. This works well for comprehensive reviews where different aspects (security, performance, testing, architecture) can be evaluated independently.

### Team Prerequisites and Fallback

Attempt to create the agent team using `TeamCreate` with a unique timestamped name: `review-pr-{number}-{YYYYMMDD-HHMMSS}` and description: "PR Review: {title}".

If team creation fails (tool unavailable or experimental features disabled), inform the user that swarm mode requires agent teams to be enabled (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json), then fall back to executing the Standard Workflow instead. The PR identifier is already parsed and ready to use.

### Phase 1: PR Information Gathering

Execute the same PR information gathering steps as Standard Workflow Phase 1 (extract PR identifier, fetch metadata, checkout branch). This establishes the shared context that all teammates will need.

After gathering PR information and checking out the branch, proceed to spawn teammates for parallel review.

### Shared Task List

Create tasks via `TaskCreate` that represent the specialized review areas:
1. **Security Review** — Identify security vulnerabilities and risks
2. **Performance Review** — Evaluate performance implications
3. **Test Coverage Review** — Assess test adequacy and quality
4. **Architecture Review** — Examine design patterns and code organization

These tasks provide structure for parallel specialized reviews.

### Teammate Roles and Spawn Protocol

After completing Phase 1, spawn 4 specialized reviewer teammates via the `Task` tool with `team_name` parameter and `subagent_type: "general-purpose"`. Each teammate prompt MUST include as string literals (NOT references to conversation):

**Required Context in Each Spawn Prompt**:
- PR title
- PR author
- Base branch name
- Head branch name
- Diff summary from `git diff $BASE_BRANCH...HEAD --stat`
- List of changed files
- For PRs with ≤50 changed files: Include the changed file contents (read them before spawning)
- For PRs with >50 changed files: Include file list only, instruct teammates to Read specific files they need to examine
- PR description/body

**Teammate 1: Security Reviewer**

```
You are conducting a security-focused review as part of a PR review team.

PR TITLE: {literal PR title}
PR AUTHOR: {literal author}
BASE BRANCH: {literal base branch}
HEAD BRANCH: {literal head branch}

DIFF SUMMARY:
{literal git diff --stat output}

CHANGED FILES:
{literal list of changed files}

[If ≤50 files]
CHANGED FILE CONTENTS:
{literal file contents}

[If >50 files]
Note: This PR has >50 changed files. Use the Read tool to examine specific files you need to review for security concerns.

PR DESCRIPTION:
{literal PR body}

YOUR TASK:
Review this PR for security vulnerabilities and risks. Focus on:
- OWASP Top 10 vulnerabilities
- Injection vulnerabilities (SQL, NoSQL, Command, LDAP, etc.)
- Authentication and authorization issues
- Secrets exposure (API keys, tokens, passwords in code)
- Dependency vulnerabilities (new or updated dependencies)
- Input validation and sanitization
- Cross-Site Scripting (XSS) potential
- Cross-Site Request Forgery (CSRF) protection
- Insecure cryptographic practices
- Security misconfiguration

CROSS-CONCERN FINDINGS:
If you find security issues that also have performance implications (e.g., DoS potential), share via SendMessage:
"Security: This {issue} at {file}:{line} also has {other concern} implications — {explanation}"

CRITICAL CONSTRAINTS:
- Use ONLY read-only tools (Read, Grep, Glob, Bash for non-destructive commands)
- DO NOT run git commit/push/add or any destructive commands
- DO NOT use AskUserQuestion (you can't interact with the user)
- DO NOT run build or test commands (may interfere with other teammates)

COMPLETION:
When you've completed your security review:
1. Mark your task complete via TaskUpdate with your findings
2. Send "REVIEW COMPLETE" via SendMessage
3. Wait for shutdown_request

FINDINGS FORMAT:
- File path and line numbers for all issues
- Severity level (Critical/High/Medium/Low)
- Specific vulnerability description
- Recommendation for remediation
```

**Teammate 2: Performance Reviewer**

```
You are conducting a performance-focused review as part of a PR review team.

PR TITLE: {literal PR title}
PR AUTHOR: {literal author}
BASE BRANCH: {literal base branch}
HEAD BRANCH: {literal head branch}

DIFF SUMMARY:
{literal git diff --stat output}

CHANGED FILES:
{literal list of changed files}

[If ≤50 files]
CHANGED FILE CONTENTS:
{literal file contents}

[If >50 files]
Note: This PR has >50 changed files. Use the Read tool to examine specific files you need to review for performance concerns.

PR DESCRIPTION:
{literal PR body}

YOUR TASK:
Review this PR for performance implications. Focus on:
- Algorithmic complexity (O(n²), O(n log n), etc.)
- Memory allocation patterns (unnecessary allocations, memory leaks)
- Database query efficiency (N+1 queries, missing indexes, overfetching)
- Caching opportunities (missing caching, cache invalidation issues)
- Bundle size impact (for frontend code)
- Async/await patterns (blocking operations, parallel opportunities)
- Resource leaks (unclosed connections, file handles, event listeners)
- Unnecessary re-renders or re-computations
- Network request optimization

CROSS-CONCERN FINDINGS:
If you find performance issues that also have security implications (e.g., DoS potential), share via SendMessage:
"Performance: This {issue} at {file}:{line} also has {other concern} implications — {explanation}"

CRITICAL CONSTRAINTS:
- Use ONLY read-only tools (Read, Grep, Glob, Bash for non-destructive commands)
- DO NOT run git commit/push/add or any destructive commands
- DO NOT use AskUserQuestion (you can't interact with the user)
- DO NOT run build or test commands (may interfere with other teammates)

COMPLETION:
When you've completed your performance review:
1. Mark your task complete via TaskUpdate with your findings
2. Send "REVIEW COMPLETE" via SendMessage
3. Wait for shutdown_request

FINDINGS FORMAT:
- File path and line numbers for all issues
- Performance impact level (Critical/High/Medium/Low)
- Specific performance concern description
- Recommendation for optimization
```

**Teammate 3: Test Coverage Reviewer**

```
You are conducting a test coverage review as part of a PR review team.

PR TITLE: {literal PR title}
PR AUTHOR: {literal author}
BASE BRANCH: {literal base branch}
HEAD BRANCH: {literal head branch}

DIFF SUMMARY:
{literal git diff --stat output}

CHANGED FILES:
{literal list of changed files}

[If ≤50 files]
CHANGED FILE CONTENTS:
{literal file contents}

[If >50 files]
Note: This PR has >50 changed files. Use the Read tool to examine specific files you need to review for test coverage.

PR DESCRIPTION:
{literal PR body}

YOUR TASK:
Review this PR for test adequacy and quality. Focus on:
- Test adequacy for all changed code paths
- Edge case coverage (boundary conditions, error states, null/undefined handling)
- Test quality (not just existence — are tests meaningful?)
- Regression potential (areas where bugs could reappear)
- Missing test scenarios (untested combinations, integration scenarios)
- Test maintainability (clear, readable, not brittle)
- Test isolation (proper mocking, no side effects)
- Assertion quality (specific assertions, not just "it doesn't crash")

CROSS-CONCERN FINDINGS:
If you find test gaps that expose security or performance risks, share via SendMessage:
"Test Coverage: Missing tests at {file}:{line} also creates {other concern} risk — {explanation}"

CRITICAL CONSTRAINTS:
- Use ONLY read-only tools (Read, Grep, Glob, Bash for non-destructive commands)
- DO NOT run git commit/push/add or any destructive commands
- DO NOT use AskUserQuestion (you can't interact with the user)
- DO NOT run build or test commands (may interfere with other teammates)

COMPLETION:
When you've completed your test coverage review:
1. Mark your task complete via TaskUpdate with your findings
2. Send "REVIEW COMPLETE" via SendMessage
3. Wait for shutdown_request

FINDINGS FORMAT:
- File path and line numbers for untested or inadequately tested code
- Coverage gap severity (Critical/High/Medium/Low)
- Specific test scenario missing
- Recommendation for test improvements
```

**Teammate 4: Architecture Reviewer**

```
You are conducting an architecture-focused review as part of a PR review team.

PR TITLE: {literal PR title}
PR AUTHOR: {literal author}
BASE BRANCH: {literal base branch}
HEAD BRANCH: {literal head branch}

DIFF SUMMARY:
{literal git diff --stat output}

CHANGED FILES:
{literal list of changed files}

[If ≤50 files]
CHANGED FILE CONTENTS:
{literal file contents}

[If >50 files]
Note: This PR has >50 changed files. Use the Read tool to examine specific files you need to review for architecture concerns.

PR DESCRIPTION:
{literal PR body}

YOUR TASK:
Review this PR for architectural quality and design patterns. Focus on:
- Design patterns (appropriate pattern usage, anti-patterns)
- SOLID principles (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion)
- Code organization (proper module/package structure, file placement)
- API design (consistent interfaces, clear contracts, backward compatibility)
- Separation of concerns (business logic vs presentation, data access layer)
- Dependency management (appropriate dependencies, circular dependencies, coupling)
- Breaking changes (API changes that affect consumers)
- Consistency with existing codebase patterns

CROSS-CONCERN FINDINGS:
If you find architectural issues that affect security, performance, or testability, share via SendMessage:
"Architecture: This {issue} at {file}:{line} also has {other concern} implications — {explanation}"

CRITICAL CONSTRAINTS:
- Use ONLY read-only tools (Read, Grep, Glob, Bash for non-destructive commands)
- DO NOT run git commit/push/add or any destructive commands
- DO NOT use AskUserQuestion (you can't interact with the user)
- DO NOT run build or test commands (may interfere with other teammates)

COMPLETION:
When you've completed your architecture review:
1. Mark your task complete via TaskUpdate with your findings
2. Send "REVIEW COMPLETE" via SendMessage
3. Wait for shutdown_request

FINDINGS FORMAT:
- File path and line numbers for all issues
- Architectural impact level (Critical/High/Medium/Low)
- Specific architectural concern description
- Recommendation for improvement
```

### Completion Protocol

Wait for all 4 teammates to signal completion by sending "REVIEW COMPLETE" messages. Timeout: 10 minutes from teammate spawn time.

**If timeout occurs**: Proceed with available findings and note which teammates timed out in the consolidated review.

**Fallback behavior**: If a teammate fails or gets stuck (repeated similar messages, no progress), you have three options:
1. Note the failure and proceed with other teammates' findings
2. Spawn a replacement teammate with clearer scoped instructions
3. Handle that review aspect yourself

Choose based on how critical that specialized review is to the overall PR assessment.

### Consolidation

As team lead, integrate teammate findings into a comprehensive PR review. Your job is to synthesize specialized findings into the same output structure used in Standard Workflow, not mechanically merge outputs.

**Reviewer Attribution**: Mark which teammate(s) found each issue: `[Security]`, `[Performance]`, `[Test Coverage]`, `[Architecture]`. Mark independently confirmed findings from multiple reviewers `[Consensus]`.

**Output Format**: Use the same review structure as Standard Workflow Phase 3:
- Executive Summary (synthesize risk level based on all reviewer findings)
- Code Quality Analysis (incorporate findings from all specialized reviews)
- Detailed File-by-File Review (merge findings by file, preserve all reviewer attributions)
- Discussion & CI Review (same as standard workflow)
- Testing Verification (same as standard workflow)
- Recommendations & Action Items (merge and prioritize findings from all reviewers)

**Cross-cutting Findings**: When findings from multiple reviewers relate to the same code location, highlight this:
"[Consensus] The dependency update at package.json:42 raises both security concerns (known CVE) and performance concerns (increased bundle size)"

### Resource Cleanup

After completing consolidation (whether successful or failed), always clean up team resources.

Send shutdown requests to all teammates via `SendMessage` with `type: "shutdown_request"`, wait briefly for confirmations, then call `TeamDelete` to remove the team and its task list.

If cleanup itself fails, inform the user: "Team cleanup incomplete. You may need to check for lingering team resources."

Execute cleanup regardless of consolidation outcome—even if earlier steps errored or teammates timed out, cleanup must run before ending.

---

## Standard Workflow

The default PR review approach for comprehensive single-agent analysis.

### Phase 1: PR Information Gathering

When the user provides a PR number or URL:

1. **Extract PR identifier**:
   - If given URL: Extract owner/repo/number
   - If given number: Use current repo context
   - If given format like "owner/repo#123": Parse accordingly

2. **Fetch PR metadata using GitHub CLI**:
   ```bash
   # Get PR details
   gh pr view {PR_NUMBER} --json title,body,state,author,headRefName,baseRefName,url,commits,reviews,comments,labels,milestone

   # Get PR diff
   gh pr diff {PR_NUMBER}

   # Get PR comments (both review comments and issue comments)
   gh pr view {PR_NUMBER} --comments

   # Get PR checks/CI status
   gh pr checks {PR_NUMBER}
   ```

3. **Checkout PR branch locally**:
   ```bash
   # Fetch the PR branch
   gh pr checkout {PR_NUMBER}

   # Get current branch name for reference
   git branch --show-current
   ```

### Phase 2: Comprehensive Analysis

1. **Compare against base branch**:
   ```bash
   # Get the base branch (usually main/master)
   BASE_BRANCH=$(gh pr view {PR_NUMBER} --json baseRefName -q .baseRefName)

   # Show diff summary
   git diff $BASE_BRANCH...HEAD --stat

   # Show full diff
   git diff $BASE_BRANCH...HEAD
   ```

2. **Analyze commit history**:
   ```bash
   # Show commits in this PR
   git log $BASE_BRANCH..HEAD --oneline --no-merges

   # Detailed commit messages
   git log $BASE_BRANCH..HEAD --no-merges
   ```

3. **Review file changes systematically**:
   - Read all changed files using Read tool
   - Pay special attention to:
     - New files (understand their purpose)
     - Deleted files (understand why removed)
     - Modified files (understand what changed and why)
     - Test files (verify test coverage)
     - Documentation (check if updated appropriately)

4. **Analyze discussion context**:
   - Review all PR comments and conversations
   - Note any unresolved discussions
   - Identify patterns in review feedback
   - Check if CI/CD checks are passing
   - Review any linked issues

### Phase 3: Generate Comprehensive Review

Provide a structured review covering:

#### 1. Executive Summary
- **PR Purpose**: Brief description of what this PR does
- **Change Scope**: High-level categorization (bugfix, feature, refactor, docs, etc.)
- **Risk Level**: Low/Medium/High based on scope and complexity
- **Recommendation**: Approve / Request Changes / Comment

#### 2. Code Quality Analysis
- **Architecture & Design**: Does it follow project patterns?
- **Code Style**: Consistent with project conventions?
- **Testing**: Adequate test coverage? Tests passing?
- **Documentation**: Inline comments, docstrings, README updates?
- **Error Handling**: Proper error handling and edge cases?
- **Performance**: Any performance implications?
- **Security**: Any security concerns?

#### 3. Detailed File-by-File Review
For each changed file:
- **File**: `path/to/file.ext`
- **Change Type**: Added/Modified/Deleted
- **Purpose**: Why this file changed
- **Review Notes**: Specific feedback
- **Issues Found**: List any problems
- **Suggestions**: Improvement recommendations

#### 4. Discussion & CI Review
- **Unresolved Conversations**: List any open threads
- **CI/CD Status**: All checks passing? Any failures?
- **Review Comments**: Summary of existing review feedback
- **Action Items**: What needs to be addressed?

#### 5. Testing Verification
If appropriate and safe:
```bash
# Run tests to verify nothing breaks
# (Only if the repo has clear test commands)
# Example: npm test, pytest, cargo test, etc.
```

#### 6. Recommendations & Action Items
Clear list of:
- **Must Fix**: Blocking issues
- **Should Fix**: Important but not blocking
- **Consider**: Nice-to-have improvements
- **Praise**: What's done well

### Phase 4: Interactive Review Session

After providing the initial review, offer to:
1. **Deep dive into specific files**: "Which file would you like me to examine more closely?"
2. **Run tests**: "Should I run the test suite to verify changes?"
3. **Check for patterns**: "Should I search for similar code patterns elsewhere in the codebase?"
4. **Draft review comment**: "Would you like me to draft a GitHub review comment?"
5. **Create follow-up tasks**: "Should I note any follow-up work needed?"

## Usage Examples

```bash
# Review a PR by number (in current repo)
/review-pr 123

# Review a PR by URL
/review-pr https://github.com/owner/repo/pull/456

# Review a PR with repo context
/review-pr owner/repo#789
```

## Important Notes

- **Branch Safety**: This command checks out the PR branch. Warn if there are uncommitted changes.
- **GitHub Authentication**: Requires `gh` CLI to be authenticated (`gh auth status`)
- **Repository Context**: Must be run from within a git repository or provide full PR URL
- **Large PRs**: For PRs with many files (>20), ask which files to prioritize
- **Private Repos**: Respects GitHub permissions via `gh` CLI authentication

## Error Handling

Handle common scenarios:
- PR doesn't exist: Verify PR number/URL
- Not authenticated: Prompt user to run `gh auth login`
- Uncommitted changes: Ask user to commit or stash first
- Merge conflicts: Note conflicts and suggest resolution
- Network issues: Suggest retry or manual `gh` command

## Output Format

Use clear markdown formatting:
- **Section headers** for organization
- `Code blocks` for commands and code snippets
- **Bold** for important findings
- Bullet lists for readability
- File paths with line references when specific: `path/to/file.py:42`

## Commit Message Convention Awareness

Check if the repo uses conventional commits (feat:, fix:, docs:, etc.) and verify PR title/commits follow the pattern.

## Follow Project Conventions

Before reviewing, check for:
- `CONTRIBUTING.md` - Review guidelines
- `CLAUDE.md` - Project-specific instructions
- `.github/PULL_REQUEST_TEMPLATE.md` - PR template requirements
- CI configuration - Understanding what checks run

Read these files first to understand project-specific review criteria.

## Rules

### Prioritization

**High Priority** (review first):
- File/line-specific changes with significant impact
- Security-related changes
- Breaking changes or API modifications
- Test coverage and quality
- Documentation completeness

**Lower Priority**:
- Style/formatting issues
- Minor refactoring
- Comment improvements

### Safety

- **No automatic git operations**: Never auto-commit or push
- **Show findings clearly**: Use structured format for easy scanning
- **Preserve context**: Understand the full picture before suggesting changes
- **Ask when unclear**: Use AskUserQuestion if anything is ambiguous

### Best Practices

- Start with understanding the PR's purpose and context
- Review commits chronologically to understand the development flow
- Check for consistency with existing codebase patterns
- Verify test coverage matches the scope of changes
- Ensure documentation is updated alongside code changes
- Look for common pitfalls: error handling, edge cases, security issues
