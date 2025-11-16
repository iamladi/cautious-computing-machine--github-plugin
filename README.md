# GitHub Plugin for Claude Code

GitHub automation plugin for CI/CD, pull requests, and code review workflows.

## Features

- 🔍 **Auto-detect CI context** - Works with PRs, feature branches, and main/master
- 📊 **Intelligent log parsing** - Identifies lint, test, type, and build errors
- 🔧 **Automated fixes** - Applies targeted fixes for common CI failures
- 🛡️ **Safety checks** - Warns about main branch changes and uncommitted work
- 🎯 **Specialized agents** - Dedicated agents for log analysis and error fixing
- 📝 **PR creation** - Generate well-formatted pull requests from GitHub issues
- 💬 **PR comment resolution** - Interactive or autonomous review comment resolution
- 🤖 **AI confidence scoring** - Smart prioritization of review comments (0-100 score)
- 🔄 **Rollback safety** - Auto-stash changes in autonomous mode for easy undo

## Installation

1. Copy this plugin to your Claude Code plugins directory:
   ```bash
   cp -r github-plugin ~/.claude/plugins/
   ```

2. Install dependencies:
   ```bash
   cd ~/.claude/plugins/github-plugin
   bun install
   ```

3. Restart Claude Code to load the plugin

## Prerequisites

- [GitHub CLI (`gh`)](https://cli.github.com/) installed and authenticated
- [Bun](https://bun.sh/) for TypeScript runtime
- Python projects: [uv](https://github.com/astral-sh/uv) for package management
- TypeScript projects: Bun already installed

## Commands

### `/fix-ci`

Auto-detect, analyze, and fix CI/CD failures.

**Usage:**
```bash
/fix-ci              # Fix CI on current branch
/fix-ci 123          # Fix CI for PR #123
/fix-ci https://...  # Fix CI for PR URL
```

**What it does:**

1. **Detects context**: Identifies whether you're on a PR, feature branch, or main branch
2. **Fetches logs**: Downloads failed CI job logs using GitHub CLI
3. **Analyzes errors**: Uses `ci-log-analyzer` agent to parse logs and identify:
   - Lint/format errors (Ruff, ESLint)
   - Test failures (pytest, jest)
   - Type errors (mypy, TypeScript)
   - Build errors (syntax, imports)
4. **Applies fixes**: Uses `ci-error-fixer` agent to automatically fix issues
5. **Reports results**: Shows diffs and completion summary

**Safety checks:**
- Warns when operating on main/master branch
- Alerts about uncommitted changes
- Flags unclear fixes for manual review
- Asks about priorities when multiple failure types exist

### `/create-pr`

Create GitHub Pull Request with proper formatting and SDLC best practices.

**Usage:**
```bash
/create-pr <issue_number>                    # Basic PR from issue
/create-pr <issue_number> path/to/plan.md    # PR with implementation plan
```

**What it does:**

1. **Fetches issue details**: Uses `gh issue view` to get issue info
2. **Generates PR title**: Format: `<issue_type>: #<issue_number> - <issue_title>`
   - Examples: `feat: #123 - Add user authentication`, `bug: #456 - Fix login error`
3. **Creates PR body with SDLC best practices**:
   - **Summary**: Explains the "why" and business context
   - **Review Focus**: Highlights architectural decisions, gotchas, and file review order
   - **Testing Notes**: Only for manual scenarios beyond CI automation
   - **References**: Links to issue, plan, and related documentation
   - Explicitly avoids redundant commit listings and generic testing documentation
4. **Reviews changes**: Shows git diff and commit history
5. **Pushes and creates**: Pushes branch and creates PR with `gh pr create`
6. **Returns PR URL**: Outputs only the PR URL for easy access

**Example:**
```bash
$ /create-pr 123 docs/implementation-plan.md

# Reviews changes, creates PR with best-practice formatting, returns:
https://github.com/user/repo/pull/456
```

**Best Practices (Enforced in Instructions):**
- ✅ Explains the "why" and business rationale
- ✅ Summarizes changes without listing every commit
- ✅ Highlights review focus: files, architecture decisions, gotchas
- ✅ Includes testing notes only for non-obvious scenarios
- ❌ No redundant commit sections (visible in PR's Commits tab)
- ❌ No generic testing documentation (CI/CD pipeline is authoritative)
- ❌ No boilerplate or placeholder sections

See `research/pr-description-best-practices.md` for research-backed guidance and authoritative sources.

### `/create-issue-from-plan`

Convert implementation plans into GitHub Issues with automatic checkboxes and linking.

**Usage:**
```bash
/create-issue-from-plan plans/feature-name.md
```

**What it does:**

1. **Parses plan file**: Reads YAML frontmatter and markdown structure
2. **Extracts content**:
   - Overview section → Issue body summary
   - Implementation Phases → Checklist items
   - Validation Commands → Issue section
   - Acceptance Criteria → Issue checklist
   - Research files → Linked in Issue body
3. **Creates Issue**: Generates GitHub Issue with:
   - Title: `<type>: <plan title>`
   - Body: Complete implementation checklist with all phases
   - Labels: `status:planning` + type label (feat, bug, chore, etc.)
4. **Updates plan file**: Adds Issue number to plan frontmatter (`issue: #123`)
5. **Commits linkage**: Saves Issue reference in plan file via git commit

**Plan Frontmatter:**
```yaml
---
title: "Feature Description"
type: Feature  # Bug|Feature|Chore|Refactor|Enhancement|Documentation
issue: null    # Auto-populated with Issue #123 after creation
research:
  - research/related-research.md
status: Draft
created: 2024-11-16
---
```

**Example:**
```bash
$ /create-issue-from-plan plans/add-oauth2-auth.md

✅ Issue #456 created from plan
📝 Updated plan frontmatter: issue: 456
🔗 Linked research files in Issue body
💾 Committed Issue linkage to plan file

Issue: https://github.com/user/repo/issues/456
```

### `/address-pr-comments`

Interactive or autonomous workflow for addressing PR review comments with AI-powered confidence scoring.

**Modes:**

The command automatically detects the execution environment:
- **Interactive Mode** (terminal): User selects which comments to address
- **Autonomous Mode** (CI/CD): Auto-addresses high-confidence comments (score ≥ 80)

**Usage:**
```bash
# Interactive mode (default in terminal)
/address-pr-comments 18
/address-pr-comments https://github.com/owner/repo/pull/123

# Autonomous mode (auto-detected in CI/CD)
/address-pr-comments 18 --autonomous
/address-pr-comments 18 auto

# Force interactive mode
/address-pr-comments 18 --interactive
```

**What it does:**

1. **Fetches PR data**: Downloads PR details, review comments, and issue comments using `gh` CLI
2. **Analyzes & scores comments**: Filters actionable comments, categorizes by type, and assigns confidence scores (0-100)
3. **Presents or filters**: Interactive shows all comments; Autonomous filters for high-confidence (≥80)
4. **Applies changes**: Reads files, explains changes, and uses Edit tool to apply fixes
5. **Reports results**: Shows detailed summary with addressed items and skipped items

**Confidence Scoring (0-100):**

Comments are scored based on:
- **Specificity** (30pts): File path + line number + concrete suggestion
- **Language clarity** (25pts): Directive ("must", "please") vs suggestive ("consider")
- **Category** (25pts): STYLE (25) > DOCS (20) > CODE (15) > TEST (10)
- **Reviewer authority** (10pts): Maintainer/owner (10) > contributor (3)
- **Discussion status** (10pts): Blocking (10) > unresolved (7)

**Thresholds:**
- **80-100 (High)**: Auto-addressed in autonomous mode
- **60-79 (Medium)**: Presented in interactive mode
- **0-59 (Low)**: Skipped or flagged for manual review

**Comment categories:**
- **CODE**: Logic changes, bug fixes, refactoring
- **STYLE**: Formatting, naming conventions
- **DOCS**: Documentation improvements
- **TEST**: Test coverage, test improvements
- **QUESTIONS**: Clarifications needed

**Safety features:**
- **Autonomous mode**: Creates git stash rollback point before changes
- No automatic commits or pushes
- Shows changes and confidence scores
- Skips ambiguous or low-confidence comments
- Detailed reporting of what was/wasn't addressed

**Interactive Mode Example:**
```bash
$ /address-pr-comments 42

Found 3 comments on PR #42: Add user authentication

1. [CODE] src/auth.ts:25 (Score: 90)
   Comment: "Add null check before accessing user.email"

2. [STYLE] src/utils.ts:42 (Score: 95)
   Comment: "Use const instead of let"

Which items would you like me to address? all

✓ Addressed item 1: Added null check
✓ Addressed item 2: Changed let to const

📊 Summary: 2 comments addressed, 2 files modified
```

**Autonomous Mode Example:**
```bash
$ /address-pr-comments 42 --auto

🤖 AUTONOMOUS MODE ACTIVE

Analyzing 5 comments on PR #42
Confidence threshold: 80/100

High-Confidence Items (2):
✓ [STYLE] src/utils.ts:42 (95) - Changed let to const
✓ [CODE] src/auth.ts:25 (90) - Added null check

Skipped Items (3):
⏭️ [CODE] src/api.ts:55 (65) - Vague suggestion
⏭️ [TEST] tests/auth.test.ts (45) - Question without action
⏭️ [QUESTIONS] src/models.ts (35) - Ambiguous feedback

🔄 Rollback: git stash apply stash^{/pre-autonomous-pr-42-...}
📊 Summary: 2 addressed, 3 skipped, 2 files modified
```

### `/review-pr`

Comprehensive PR review with detailed analysis and actionable feedback.

**Usage:**
```bash
/review-pr 123                              # Review PR by number
/review-pr https://github.com/owner/repo/pull/456  # Review by URL
/review-pr owner/repo#789                   # Review with repo context
```

**What it does:**

1. **Fetches PR data**: Downloads PR details, diff, comments, and CI status using `gh` CLI
2. **Checks out branch**: Switches to PR branch for local analysis
3. **Analyzes changes**: Reviews all modified files, commit history, and diff
4. **Reviews discussions**: Examines comments, unresolved threads, and CI results
5. **Generates report**: Provides structured review with executive summary, code quality analysis, file-by-file feedback

**Review output includes:**
- **Executive Summary**: Purpose, scope, risk level, recommendation (Approve/Request Changes)
- **Code Quality Analysis**: Architecture, style, testing, documentation, error handling, security
- **File-by-File Review**: Detailed feedback for each changed file
- **Discussion Review**: Unresolved conversations, CI status, action items
- **Recommendations**: Must fix, should fix, consider, and praise items

**Safety checks:**
- Warns about uncommitted changes before checkout
- Requires GitHub CLI authentication
- Respects repository permissions
- Checks project conventions (CONTRIBUTING.md, PR templates)

## Agents

### `ci-log-analyzer`

Specialized agent for parsing CI logs and identifying error patterns.

**Capabilities:**
- Recognizes 10+ error pattern types
- Extracts file paths, line numbers, and error messages
- Categorizes by type and severity
- Returns structured JSON for automated fixing

**Tools:** Read, Grep, Bash

### `ci-error-fixer`

Specialized agent for applying fixes based on error types.

**Capabilities:**
- Applies targeted fixes for each error category
- Shows diffs for all changes
- Flags complex issues for manual review
- Respects code style and best practices

**Tools:** Read, Edit, Write, Bash

## Supported Error Types

| Category | Examples | Fix Strategy |
|----------|----------|--------------|
| **Lint** | Ruff formatting, unused imports, ESLint errors | Auto-format, remove unused code |
| **Test** | pytest/jest failures, assertion errors, missing imports | Fix logic, add imports, update tests |
| **Type** | mypy/TypeScript type mismatches, missing returns | Add type hints, fix type errors |
| **Build** | Syntax errors, import errors, compilation failures | Fix syntax, resolve imports |

## Example Workflow

```bash
# You're on a feature branch with failing CI
$ /fix-ci

# Plugin detects context
✓ Detected feature branch: feature/user-auth
✓ Found PR #42: "Add user authentication"
✓ Identified 3 failed checks

# Fetches and analyzes logs
✓ Analyzing CI logs...
  Found 5 errors: 2 lint, 2 test, 1 type

# Applies fixes
✅ Fixed: src/auth.py:15 - Removed unused import 'os'
✅ Fixed: src/auth.py:42 - Added missing type hint
✅ Fixed: tests/test_auth.py:10 - Added missing import 'requests'
⚠️  Flagged: tests/test_auth.py:55 - Assertion failure needs review

# Summary
🎉 Fix Summary
  ✅ 3 issues fixed
  ⚠️  1 issue flagged for manual review

Next steps:
  1. Review changes: git diff
  2. Run tests locally: pytest
  3. Commit: git add . && git commit -m "fix: CI failures"
  4. Push: git push
```

## Development

### Validate Plugin

```bash
bun run validate
```

### Run Tests

```bash
# Test plugin manifest validation
bun run validate

# Test in Claude Code
# Copy to plugins directory and restart Claude Code
```

### Contributing

1. Make changes to commands or agents
2. Run validation: `bun run validate`
3. Test in Claude Code
4. Update CHANGELOG.md
5. Submit PR

## Project Structure

```
github-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── .github/workflows/
│   └── validate-plugin.yml      # CI validation
├── commands/
│   ├── fix-ci.md                # CI fixing command
│   ├── create-pr.md             # PR creation command
│   ├── address-pr-comments.md   # PR comment resolution
│   └── review-pr.md             # PR review command
├── agents/
│   ├── ci-log-analyzer.md       # Log parsing agent
│   └── ci-error-fixer.md        # Error fixing agent
├── scripts/
│   └── validate-plugin.ts       # Validation script
├── package.json
├── README.md
├── CHANGELOG.md
└── LICENSE
```

## Troubleshooting

### "gh: command not found"

Install GitHub CLI:
```bash
# macOS
brew install gh

# Linux
sudo apt install gh

# Windows
winget install GitHub.cli
```

### "Not authenticated with GitHub"

Authenticate:
```bash
gh auth login
```

### "No failed CI runs found"

Check that CI has actually run:
```bash
gh run list --limit 5
```

### Plugin not loading

1. Verify plugin is in `~/.claude/plugins/github-plugin/`
2. Check plugin.json is valid: `bun run validate`
3. Restart Claude Code

## License

MIT License - see [LICENSE](LICENSE) file for details

## Author

Ladislav Martincik ([@iamladi](https://github.com/iamladi))

## Links

- [Claude Code Documentation](https://docs.claude.com/en/docs/claude-code)
- [GitHub CLI Documentation](https://cli.github.com/manual/)
- [Plugin Repository](https://github.com/iamladi/claude-code-plugins)
