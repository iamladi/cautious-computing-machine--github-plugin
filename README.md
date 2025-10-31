# GitHub Plugin for Claude Code

GitHub CI/CD automation plugin that auto-detects, analyzes, and fixes CI/CD failures on any branch.

## Features

- 🔍 **Auto-detect CI context** - Works with PRs, feature branches, and main/master
- 📊 **Intelligent log parsing** - Identifies lint, test, type, and build errors
- 🔧 **Automated fixes** - Applies targeted fixes for common CI failures
- 🛡️ **Safety checks** - Warns about main branch changes and uncommitted work
- 🎯 **Specialized agents** - Dedicated agents for log analysis and error fixing

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
│   └── fix-ci.md                # Main CI fixing command
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
