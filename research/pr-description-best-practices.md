---
date: 2025-11-16T11:00:00+01:00
git_commit: cffb4ed5135ec12c4d38a48c10b53793077dca88
branch: main
repository: github-plugin
topic: "PR Description Best Practices for SDLC and LLM-Generated PRs"
tags: [research, sdlc, pr-best-practices, quality-standards, github]
status: complete
last_updated: 2025-11-16
last_updated_by: HAL
---

# Research: PR Description Best Practices for SDLC

**Date**: November 16, 2025
**Git Commit**: cffb4ed5135ec12c4d38a48c10b53793077dca88
**Branch**: main
**Repository**: github-plugin

## Research Question
What constitutes a high-quality PR description according to SDLC best practices, and what sections should be included vs. excluded to avoid the common pitfalls of LLM-generated PRs?

## Summary
Industry best practices (GitHub, FreeCodeCamp, AWS, enterprise SDLC standards) converge on a clear principle: **PR descriptions should communicate the "why" and "what," not replicate information already available elsewhere.** The current `create-pr` command correctly identifies the key valuable sections but lacks explicit guidance on what NOT to include.

Key insight: **Commits don't belong in PR descriptions** because the full commit history is already visible in the PR's commit tab. **Manual testing documentation doesn't belong** because modern SDLC relies on CI/CD automation. LLM PRs are criticized when they waste reviewer time with information duplication.

## Detailed Findings

### What SHOULD Be Included in PR Descriptions

According to authoritative sources (GitHub official docs, GitHub blog, FreeCodeCamp):

#### 1. **Purpose & Context (The "Why")**
- Explain the business or technical reason the change was made
- Don't assume reviewers have historical context
- Link to tracking issues (Closes #123 format)
- Provide overview of goals the change helps achieve
- Example: "This implements user authentication required for the onboarding flow (see #456)"

#### 2. **Summary of Changes (The "What")**
- High-level description of what was modified
- Scope of changes: code, configuration, migrations, APIs
- Deployment impact indicators
- **Key point**: Summarize, don't list every commit
- Reference: `commands/create-pr.md:24` - already includes "summary section explaining the why and context"

#### 3. **Review Guidance**
- Specify what type of feedback is needed (code review, design critique, technical discussion)
- Highlight files to review and order if complex
- Call out potential gotchas or architectural decisions
- Explicitly state level of review needed
- Reference: `commands/create-pr.md:27` - "Review Focus" section addressing this

#### 4. **Reference Links & Documentation**
- Link to original issue/ticket (automatic closure with keywords)
- Reference related PRs or previous context
- Attach relevant documents: requirements, diagrams, design patterns
- Reference: `commands/create-pr.md:26` - already includes issue reference

#### 5. **Testing Notes (Only When Non-Obvious)**
- Include ONLY if testing involves workflows beyond automated CI checks
- Describe scenarios requiring special attention
- Detail flows, APIs, crons, workers needing verification
- **Key point**: Don't document what CI tests cover
- Reference: `commands/create-pr.md:28` - correctly states "only if there's something non-obvious beyond CI checks"

#### 6. **Dependencies & Configuration** (If Applicable)
- Specify dependent repositories
- Required config changes (without exposing secrets)
- Breaking changes or migration requirements

### What Should NOT Be Included in PR Descriptions

This is where LLM-generated PRs commonly fail. Based on best practices analysis:

#### ❌ **Commit List/History**
- **Why it's waste**: Full commit history is already visible in the PR's "Commits" tab
- **Problem**: Duplicates information and makes description harder to scan
- **Better approach**: Summarize changes into a coherent narrative
- **User complaint in this repo**: "Commits section is pointless to have it in description"

#### ❌ **Manual Testing/Testing Results**
- **Why it's waste**: Modern SDLC relies on CI/CD automation for test coverage
- **Problem**: Creates false sense that manual testing was documented in PR, when CI handles it
- **Better approach**: Reference CI pipeline status; only document exceptions (manual scenarios CI can't cover)
- **User complaint in this repo**: "We have automated tests, so there's no need to write this, instead we rely on the CI/CD pipeline"

#### ❌ **Generic/Boilerplate Sections**
- Section headers without substance
- Placeholder text like "Testing: Yes"
- Filler that doesn't clarify anything new

#### ❌ **Assumptions About Reviewer Context**
- Don't assume historical knowledge of requirements
- Don't reference internal conversations without context
- Always link to external documentation

#### ❌ **Implementation Details That Belong in Code Comments**
- Line-by-line code explanations (belong in code as comments)
- Pseudo-code or algorithm descriptions (belong in docstrings)
- The PR description is for reviewers, not code documentation

### Current Implementation Analysis

**File**: `commands/create-pr.md` (lines 23-28)

Current PR body structure:
```
- A summary section explaining the **why** and context
- Link to the implementation plan file
- Reference to the issue (Closes #<issue_number>)
- "Review Focus" section highlighting what reviewers should pay attention to, potential gotchas, or architectural decisions
- "Testing Notes" section (only if there's something non-obvious beyond CI checks)
```

**Assessment**: The structure is **already aligned with best practices.** The command correctly:
- ✅ Emphasizes "why" and context
- ✅ Includes Review Focus for architectural clarity
- ✅ Limits Testing Notes to non-obvious scenarios
- ✅ References the issue for closure
- ⚠️ Mentions implementation plan link (verify this adds value vs. redundancy)

**What's missing**: No explicit instruction to EXCLUDE commits, which may be why current PR descriptions include them.

### Industry Standard Structure (What LLM PRs Should Generate)

Based on GitHub's official guidelines and enterprise best practices:

```
## PR Title Format
<type>: #<issue_number> - <brief_description>

## PR Body Structure

### Purpose
Explain the "why" - business reason, goals, linked issue

### What Changed
Brief summary of changes (code, config, migrations, APIs)

### Review Focus
- Files to prioritize
- Architectural decisions made
- Potential gotchas
- Special considerations

### Testing Notes (If Non-Obvious)
- Manual test scenarios CI can't cover
- Special workflows to verify

### References
- Closes #<issue>
- Related PRs
- External documentation links
```

### Common LLM PR Mistakes to Avoid

1. **Listing commits as separate section** ← Your current issue
2. **Documenting all test coverage** ← Your current issue
3. **Over-explaining obvious code logic**
4. **Creating "Changes" section that mirrors commit messages**
5. **Including unverified assumptions about reviewer knowledge**
6. **Making PR description longer than the actual code changes**
7. **Separating "What" from "Why" into disconnected sections**

## Code References

### Primary Implementation
- `commands/create-pr.md:16-22` - PR title format instructions
- `commands/create-pr.md:23-28` - PR body structure definition
- `commands/create-pr.md:34` - Execution using `gh pr create` command

### Supporting Files
- `.claude-plugin/plugin.json` - Plugin manifest registering create-pr command
- `README.md` - High-level command documentation
- `CHANGELOG.md` - Feature documentation

## Architecture & Design

**Key Design Pattern**: The `create-pr` command is a **prompt-based plugin** that works through interpreted markdown instructions. There is no separate template file or code generation logic—Claude interprets the natural language instructions in `create-pr.md` to construct the PR body dynamically.

This approach means:
- Changes to PR structure are centralized in one file
- The "instructions" serve as both documentation and specification
- Clarity in instructions directly impacts quality of generated PRs

## Authoritative Sources

- **GitHub Official Documentation**: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/getting-started/best-practices-for-pull-requests
- **GitHub Blog**: https://github.blog/2015-01-21-how-to-write-the-perfect-pull-request/
- **FreeCodeCamp Guide**: https://www.freecodecamp.org/news/how-to-write-a-pull-request-description/
- **Enterprise Best Practices**: https://cloudcity.io/blog/2022/08/08/what-to-include-in-a-pull-request-description/

## Recommendations for Implementation

To ensure generated PRs follow SDLC best practices and avoid common LLM mistakes:

1. **Add explicit exclusion rules** to `create-pr.md` instructions:
   - "DO NOT include a separate 'Commits' section—commit history is visible in the PR tab"
   - "DO NOT document test coverage—rely on CI/CD pipeline results"

2. **Strengthen Review Focus section** guidance to encourage:
   - File review order (especially for complex PRs)
   - Architecture decisions that might need discussion
   - Edge cases or potential issues

3. **Clarify Testing Notes scope** by adding:
   - "Only include scenarios that require manual verification beyond automated CI checks"
   - "Example: user acceptance testing, performance benchmarking in staging, multi-browser testing"

4. **Consider a maximum length guideline**:
   - PR descriptions that are longer than the feature code should be condensed
   - Forces focus on what truly matters

## Summary for Quick Reference

| Should Include | Should Exclude |
|---|---|
| Purpose & context (the "why") | Commit list/history |
| Summary of changes (the "what") | Manual test documentation |
| Review focus & gotchas | Boilerplate sections |
| Testing notes (if non-obvious) | Implementation details (code comments are better) |
| Issue reference & links | Generic placeholder text |
| | Unverified assumptions about context |

---

**Status**: Complete and ready for implementation review
**Next Steps**: Use these findings to enhance `create-pr.md` instructions with explicit exclusion rules
