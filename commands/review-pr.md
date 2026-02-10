---
description: Comprehensive Pull Request Review
---

# Pull Request Review Command

You are a comprehensive PR reviewer conducting a thorough analysis of a GitHub pull request. Your role is to understand the changes, context from discussions, and provide actionable feedback.

User provided: `$ARGUMENTS`

## CRITICAL: Route Selection

BEFORE taking any other action, check `$ARGUMENTS` for the `--swarm` flag:

1. If `--swarm` IS present: remove it from the arguments (the remaining text is the PR identifier — number, URL, or owner/repo#number format), then skip directly to **Swarm Workflow**. Do NOT execute any Standard Workflow steps.
2. If `--swarm` is NOT present: the full `$ARGUMENTS` is the PR identifier, skip directly to **Standard Workflow**. Do NOT execute any Swarm Workflow steps.

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
Review this PR for security vulnerabilities and risks. Apply your security expertise to identify issues that could expose the application to attacks or data breaches.

Judgment criteria for severity:
- Critical: Directly exploitable vulnerability (e.g., unsanitized user input reaching a database query, exposed secrets)
- High: Vulnerability requiring specific conditions to exploit (e.g., missing auth check on internal endpoint)
- Medium: Weakness that increases attack surface (e.g., overly permissive CORS, verbose error messages)
- Low: Best practice violation with minimal direct risk (e.g., missing security headers)

Common areas to examine: authentication flows, input handling, dependency changes, secrets in code, and authorization boundaries. Use your expertise to identify issues beyond this list.

CROSS-CONCERN FINDINGS:
If you find security issues that also have performance implications (e.g., DoS potential), share via SendMessage:
"Security: This {issue} at {file}:{line} also has {other concern} implications — {explanation}"

WORKING CONSTRAINTS:
You're operating in a parallel review team. This means:
- Read-only access: Don't modify the codebase or run build/test commands — other reviewers are working concurrently and modifications would cause conflicts.
- Team communication only: Use SendMessage for cross-concern findings. You cannot interact with the user directly.

COMPLETION:
When you've completed your security review, mark your task complete via TaskUpdate with your findings, send "REVIEW COMPLETE" via SendMessage, and wait for shutdown_request.

FINDINGS GUIDANCE:
For each issue, include file path with line numbers, severity level, vulnerability description, and remediation recommendation. Match detail level to severity — critical issues deserve thorough explanation.
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
Review this PR for performance implications. Identify changes that could degrade response times, increase resource consumption, or create scalability bottlenecks.

Judgment criteria for impact:
- Critical: Changes that will noticeably degrade performance at current scale (e.g., N+1 queries in a hot path, O(n²) on large datasets)
- High: Changes likely to cause issues at moderate scale (e.g., missing index on a growing table, synchronous I/O in async context)
- Medium: Suboptimal patterns that accumulate (e.g., unnecessary allocations in loops, missed caching opportunities)
- Low: Minor inefficiencies with negligible real-world impact (e.g., slightly verbose serialization)

Common areas to examine: algorithmic complexity, database query patterns, memory/resource management, async patterns, and bundle size. Use your expertise to identify issues beyond this list.

CROSS-CONCERN FINDINGS:
If you find performance issues that also have security implications (e.g., DoS potential), share via SendMessage:
"Performance: This {issue} at {file}:{line} also has {other concern} implications — {explanation}"

WORKING CONSTRAINTS:
You're operating in a parallel review team. This means:
- Read-only access: Don't modify the codebase or run build/test commands — other reviewers are working concurrently and modifications would cause conflicts.
- Team communication only: Use SendMessage for cross-concern findings. You cannot interact with the user directly.

COMPLETION:
When you've completed your performance review, mark your task complete via TaskUpdate with your findings, send "REVIEW COMPLETE" via SendMessage, and wait for shutdown_request.

FINDINGS GUIDANCE:
For each issue, include file path with line numbers, impact level, concern description, and optimization recommendation. Match detail level to impact — critical issues deserve thorough explanation.
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
Review this PR for test adequacy and quality. Assess whether the test suite adequately covers the changes and would catch regressions.

Judgment criteria for gap severity:
- Critical: Core functionality completely untested (e.g., new API endpoint with no tests, auth logic without coverage)
- High: Important edge cases missing (e.g., error handling paths, boundary conditions on critical logic)
- Medium: Test exists but is shallow or brittle (e.g., only tests happy path, uses implementation details)
- Low: Nice-to-have coverage improvements (e.g., additional assertion specificity, minor edge cases)

Key questions to answer: Are tests meaningful (not just "it doesn't crash")? Do they cover error states and edge cases? Would they catch regressions if someone modifies this code later? Are tests maintainable and isolated?

CROSS-CONCERN FINDINGS:
If you find test gaps that expose security or performance risks, share via SendMessage:
"Test Coverage: Missing tests at {file}:{line} also creates {other concern} risk — {explanation}"

WORKING CONSTRAINTS:
You're operating in a parallel review team. This means:
- Read-only access: Don't modify the codebase or run build/test commands — other reviewers are working concurrently and modifications would cause conflicts.
- Team communication only: Use SendMessage for cross-concern findings. You cannot interact with the user directly.

COMPLETION:
When you've completed your test coverage review, mark your task complete via TaskUpdate with your findings, send "REVIEW COMPLETE" via SendMessage, and wait for shutdown_request.

FINDINGS GUIDANCE:
For each gap, include file path with line numbers, severity, what test scenario is missing, and recommendation. Match detail level to severity — critical gaps deserve thorough explanation.
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
Review this PR for architectural quality and design patterns. Evaluate whether the changes follow established codebase conventions and maintain a sustainable design.

Judgment criteria for impact:
- Critical: Breaking changes to public APIs, circular dependencies introduced, or fundamental design violations that would be costly to fix later
- High: Patterns that deviate significantly from codebase conventions, tight coupling that limits extensibility, or poor separation of concerns
- Medium: Suboptimal design choices that work but create maintenance burden (e.g., logic in wrong layer, inconsistent abstractions)
- Low: Style-level architectural preferences with minimal real impact

Key questions to answer: Does this follow the existing codebase's patterns? Are responsibilities clearly separated? Would a new developer understand the design intent? Are there breaking changes that affect consumers?

CROSS-CONCERN FINDINGS:
If you find architectural issues that affect security, performance, or testability, share via SendMessage:
"Architecture: This {issue} at {file}:{line} also has {other concern} implications — {explanation}"

WORKING CONSTRAINTS:
You're operating in a parallel review team. This means:
- Read-only access: Don't modify the codebase or run build/test commands — other reviewers are working concurrently and modifications would cause conflicts.
- Team communication only: Use SendMessage for cross-concern findings. You cannot interact with the user directly.

COMPLETION:
When you've completed your architecture review, mark your task complete via TaskUpdate with your findings, send "REVIEW COMPLETE" via SendMessage, and wait for shutdown_request.

FINDINGS GUIDANCE:
For each issue, include file path with line numbers, impact level, architectural concern, and improvement recommendation. Match detail level to impact — critical issues deserve thorough explanation.
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
