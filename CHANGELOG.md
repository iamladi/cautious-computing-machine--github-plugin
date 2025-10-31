# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
