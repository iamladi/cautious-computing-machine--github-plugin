# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.3.0] - 2024-11-16

### Added

- **New `/github:create-issue-from-plan` command** - Converts plans into GitHub Issues
  - Parses plan YAML frontmatter (title, type, research, etc.)
  - Extracts Implementation Phases as checklist items
  - Creates Issue with plan Overview as summary
  - Includes Validation Commands section
  - Links research files from plan frontmatter
  - Updates plan frontmatter with Issue number after creation
  - Commits Issue linkage back to plan file

### Enhanced

- `/create-pr` command now extracts and links plan information
  - Extracts plan type from frontmatter for PR labels
  - Links plan file in PR body: `See plans/file.md for full design`
  - Includes research file links from plan metadata
  - Generates Review Focus from Implementation Plan phases
  - Automatically references Testing Strategy from plan
  - Generates PR title with plan type: `<type>: #<issue> - Title`

### Benefits

- Complete Plan → Issue → PR traceability chain
- Plans drive Issue creation automatically
- PR body automatically linked to source documentation
- Single source of truth: Plan = specification
- Reduced manual linking and documentation duplication

## [1.2.1] - 2024-11-16

### Enhanced

- `/create-pr` command instructions now enforce SDLC best practices
  - **Explicit "DO NOT include" rules** to prevent common LLM-generated PR mistakes
    - Prevents redundant commit listings (already visible in PR's Commits tab)
    - Prevents manual testing documentation (CI/CD automation is the source of truth)
    - Prevents boilerplate and placeholder sections
  - **Detailed Review Focus guidance** with concrete examples
    - Specifies files to review in priority order
    - Highlights architectural decisions and gotchas
    - Shows what adds value vs. placeholder text
  - **Clarified Testing Notes scope** with examples
    - Only include scenarios beyond CI automation (UAT, multi-browser, performance)
    - Examples of what is/isn't "non-obvious"
  - **Scope guidelines** to keep descriptions concise
    - PR descriptions should be shorter than code changes
    - Forces focus on truly important information
  - See `research/pr-description-best-practices.md` for authoritative sources and deeper context

## [1.2.0] - 2025-10-31

### Added

- `/address-pr-comments` command for interactive or autonomous PR comment resolution
  - **Dual-mode operation**: Interactive (terminal) and Autonomous (CI/CD)
  - **AI-powered confidence scoring** (0-100) for comment prioritization
    - Specificity scoring: file path + line number + concrete suggestions
    - Language clarity: directive vs suggestive language detection
    - Category confidence: STYLE (25) > DOCS (20) > CODE (15) > TEST (10)
    - Reviewer authority: maintainer/owner weight higher than contributors
    - Discussion status: blocking issues prioritized
  - **Autonomous mode features**:
    - Auto-detects CI/CD environment (GITHUB_ACTIONS, CI, JENKINS_URL, etc.)
    - Filters high-confidence comments (score ≥ 80) for automatic resolution
    - Creates git stash rollback point before any changes
    - Detailed reporting of addressed and skipped items with reasoning
  - **Interactive mode features**:
    - User selects which comments to address
    - Shows confidence scores for all comments
    - Comprehensive explanations and diffs
  - Fetches PR review comments and issue comments using gh CLI
  - Analyzes and categorizes comments by type (CODE, STYLE, DOCS, TEST, QUESTIONS)
  - Applies changes with Edit tool and shows diffs
  - Safety features: no auto-commits, warns about uncommitted changes
  - Support for mode override: `--autonomous`, `--auto`, or `--interactive` flags
- Keywords: "code-review", "review-comments", "autonomous", "ai-review", "confidence-scoring"

### Changed

- Plugin description updated to include code review and autonomous functionality
- README updated with comprehensive `/address-pr-comments` documentation
  - Dual-mode operation explained
  - Confidence scoring algorithm documented
  - Interactive and autonomous mode examples
  - Safety features and rollback instructions

## [1.1.0] - 2025-10-31

### Added

- `/create-pr` command for creating GitHub Pull Requests with proper formatting
  - Fetches issue details from GitHub
  - Generates structured PR title in format: `<type>: #<number> - <title>`
  - Creates comprehensive PR body with summary, plan link, issue reference, and changes
  - Supports optional implementation plan file as second argument
  - Returns PR URL for easy access
- Keywords: "pull-request", "pr" for better discoverability

### Changed

- Plugin description updated to include PR creation functionality

## [1.0.0] - 2025-10-31

### Added

- Initial release of GitHub CI/CD automation plugin
- `/fix-ci` command for auto-detecting, analyzing, and fixing CI/CD failures
- `ci-log-analyzer` agent for parsing CI logs and identifying error patterns
  - Supports lint errors (Ruff, ESLint)
  - Supports test failures (pytest, jest)
  - Supports type errors (mypy, TypeScript)
  - Supports build errors (syntax, imports)
- `ci-error-fixer` agent for applying targeted fixes
  - Auto-format code
  - Fix imports and type hints
  - Update test assertions
  - Fix syntax errors
- Safety checks for main branch operations and uncommitted changes
- Support for multiple invocation patterns (current branch, PR number, PR URL)
- Comprehensive documentation and examples
- GitHub Actions workflow for plugin validation
- Zod-based plugin manifest validation

### Features

- Auto-detect CI context (PR, feature branch, main/master)
- Intelligent log parsing with pattern recognition
- Structured error categorization by type and severity
- Automated fixes with diff display
- Manual review flagging for complex issues
- Priority handling for multiple failure types

[1.0.0]: https://github.com/iamladi/claude-code-plugins/releases/tag/github-plugin-v1.0.0
