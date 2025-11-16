# PRD: Enhance create-pr Command to Enforce SDLC Best Practices

## Metadata
- **Type**: Enhancement
- **Priority**: High
- **Severity**: Major (PR quality issue affecting reviewers)
- **Estimated Complexity**: 3/10
- **Created**: 2025-11-16
- **Status**: Ready for Implementation

## Overview

### Problem Statement
The `/create-pr` command's instructions are well-structured and aligned with industry best practices, but lack **explicit exclusion rules** to prevent LLM-generated PRs from including unnecessary information. Specifically:

1. **Commit listing**: Current instructions don't explicitly forbid including a separate "Commits" section, which duplicates the PR's native "Commits" tab and wastes reviewer time
2. **Testing documentation**: No clear guidance that manual testing details shouldn't be documented, since modern SDLC relies on CI/CD automation
3. **Vague review guidance**: "Review Focus" section could be more prescriptive about what constitutes valuable guidance vs. boilerplate
4. **No scope guardrails**: No length guidelines or clarity on what constitutes "implementation details" that belong in code comments, not PR descriptions

Users have explicitly complained about these issues:
- "Commits (section) is pointless to have it in description"
- "We have automated tests, so there's no need to write this, instead we rely on the CI/CD pipeline"

This directly contradicts industry best practices documented in research: GitHub, FreeCodeCamp, AWS, and enterprise SDLC standards all agree that PR descriptions should communicate "why" and "what," not duplicate information already visible elsewhere.

### Goals & Objectives

1. Add explicit "DO NOT include" rules to prevent common LLM PR mistakes
2. Strengthen guidance for valuable Review Focus sections with concrete examples
3. Clarify Testing Notes scope with specific scenarios that belong vs. don't belong
4. Implement length/scope guidelines to prevent PR descriptions from becoming longer than the code changes
5. Ensure all generated PRs follow SDLC best practices and minimize reviewer friction

### Success Metrics

- **Primary Metric**: Generated PR descriptions no longer include redundant commit lists or manual testing documentation
- **Secondary Metrics**:
  - Review Focus sections include actionable architectural guidance
  - Testing Notes only document scenarios beyond CI/CD scope
  - PR descriptions remain concise (shorter than code changes)
- **Quality Gates**:
  - All existing PR structures still valid (backward compatible)
  - Instructions are clear and unambiguous
  - Examples demonstrate desired behavior

## User Stories

### Story 1: Prevent Redundant Commit Documentation
- **As a**: Code reviewer
- **I want**: PR descriptions that don't list commits (already visible in PR tab)
- **So that**: I can focus on meaningful context instead of scrolling through duplicated information
- **Acceptance Criteria**:
  - [ ] Instructions explicitly forbid separate "Commits" section
  - [ ] Examples show correct summary without listing commits
  - [ ] Guidance explains why (duplication with PR's Commits tab)

### Story 2: Skip Obvious Testing Information
- **As a**: Code reviewer
- **I want**: PR descriptions that don't document what CI tests cover
- **So that**: I can trust the CI pipeline and focus on non-obvious testing scenarios
- **Acceptance Criteria**:
  - [ ] Instructions clarify Testing Notes are "only if non-obvious" (current language preserved)
  - [ ] Examples provided of what is/isn't non-obvious
  - [ ] Guidance explains CI/CD is the source of truth

### Story 3: Get Valuable Review Guidance
- **As a**: Code reviewer
- **I want**: Review Focus sections that highlight real gotchas and architectural decisions
- **So that**: I know where to focus deep review effort and what the author is concerned about
- **Acceptance Criteria**:
  - [ ] Instructions specify files to review and order (if complex)
  - [ ] Examples show architectural decisions that matter
  - [ ] Guidance distinguishes valuable context from placeholder text

## Requirements

### Functional Requirements

1. **FR-1**: Add explicit exclusion rules to `create-pr.md` instructions
   - Details: Expand instructions (lines 23-28) to include "DO NOT" guidance
   - Sections: Commits, testing, boilerplate, implementation details
   - Priority: Must Have

2. **FR-2**: Strengthen Review Focus section guidance
   - Details: Provide concrete examples of valuable Review Focus content
   - Include: File review order, architecture decisions, gotchas
   - Priority: Must Have

3. **FR-3**: Clarify Testing Notes scope with examples
   - Details: Define what is/isn't "non-obvious beyond CI checks"
   - Examples: UAT, performance benchmarking, multi-browser testing, special workflows
   - Priority: Must Have

4. **FR-4**: Add length/scope guidelines
   - Details: Maximum length guidance to prevent bloat
   - Rationale: Forces focus on what truly matters
   - Priority: Should Have

5. **FR-5**: Preserve implementation plan link guidance
   - Details: Maintain current instruction about plan file link
   - Rationale: Verify this adds value vs. redundancy (per research)
   - Priority: Should Have

### Non-Functional Requirements

1. **NFR-1**: Backward Compatibility
   - Requirement: All existing PR structures remain valid
   - Target: 100% compatibility with current workflows
   - Measurement: No breaking changes to command behavior

2. **NFR-2**: Clarity & Conciseness
   - Requirement: Instructions must be easy to follow
   - Target: Single-read comprehensibility
   - Measurement: Clear examples for each section

3. **NFR-3**: Alignment with Industry Standards
   - Requirement: Follow GitHub, FreeCodeCamp, enterprise best practices
   - Target: Reference authoritative sources
   - Measurement: Sourced from research document

## Scope

### In Scope
- Enhance instructions in `commands/create-pr.md` (lines 14-28)
- Add "DO NOT include" section with specific examples
- Strengthen Review Focus and Testing Notes guidance
- Update example PR titles/body if needed
- Add cross-reference to research document

### Out of Scope
- Change command behavior or execution logic
- Create separate template file (instruction-based approach is better)
- Modify other commands (fix-ci, review-pr, address-pr-comments)
- Change PR title format or overall structure
- Implement automated PR validation

### Future Considerations
- Automated PR description validation/linting (post-generation)
- CI/CD checks to flag PR descriptions with commits sections
- ML-based review of generated PR quality
- Plugin configuration for custom PR templates

## Impact Analysis

### Affected Areas
- `/create-pr` command (instructions only)
- Plugin documentation (README.md references)
- CHANGELOG.md (feature enhancement note)

### Users Affected
- Users running `/create-pr` command
- Code reviewers reading generated PRs
- LLM assistant (Claude) interpreting instructions

### System Impact
- **Performance**: No impact (instruction-only changes)
- **Security**: No impact (no new data processing)
- **Data Integrity**: No impact (no data model changes)

### Dependencies
- Depends on: Research document for authoritative guidance
- Upstream: None
- Downstream: All `/create-pr` command executions

### Breaking Changes
- [x] **None** - This is a backward-compatible enhancement

## Root Cause Analysis

Why are LLM-generated PRs criticized?

1. **Why** do LLMs include commits sections?
   - Current instructions don't explicitly forbid it
   - LLMs try to be thorough and include all available info

2. **Why** do current instructions lack this guidance?
   - Command was created correctly for human use
   - Gaps only became apparent when used with AI generation

3. **Why** is this a problem?
   - Wastes reviewer time with redundant information
   - Makes PRs look like they're from LLMs (not polished)
   - Contradicts SDLC best practices

4. **Why** not just fix in the model?
   - Instructions are the right place for behavior specification
   - Clearer instructions = better AI outputs
   - Documentation serves as source of truth for all users

5. **Root Cause**: Insufficient explicit guidance in `create-pr.md` instructions about what NOT to include

## Solution Design

### Approach
Enhance `commands/create-pr.md` instructions (lines 14-28) with:

1. **Explicit DO NOT rules** - Clear list of what to exclude
2. **Detailed guidance for each section** - Expand each bullet point with rationale and examples
3. **Concrete examples** - Show good vs. bad PR descriptions
4. **Length guideline** - Optional maximum for description length
5. **Cross-reference** - Link to research document for deeper context

The instruction file serves as both command specification and documentation, so improvements to clarity directly improve PR generation quality.

### Alternatives Considered

1. **Alternative 1**: Create separate template file
   - Pros: Decoupled template from logic
   - Cons: Extra file to maintain; instruction-based approach is cleaner
   - Why rejected: Current approach (interpreted instructions) is more flexible

2. **Alternative 2**: Implement PR validation in CI
   - Pros: Catches issues post-generation
   - Cons: Requires more implementation; better to prevent at source
   - Why rejected: Out of scope; instruction improvement is more direct

3. **Alternative 3**: Modify Claude model instructions
   - Pros: System-level improvement
   - Cons: Can't control external model behavior
   - Why rejected: Not viable; instructions in command are the right lever

### Data Model Changes
None - instruction-only changes

### API Changes
None - command behavior unchanged

### UI/UX Changes
None - command interface unchanged

## Implementation Plan

### Phase 1: Document Enhancement
**Complexity**: 2/10 | **Priority**: High

- [x] Read current `commands/create-pr.md` (already done in research)
- [x] Create enhanced instructions with explicit DO NOT rules
- [x] Add detailed guidance for Review Focus section
- [x] Add detailed guidance for Testing Notes section
- [x] Include concrete good/bad examples
- [x] Add optional length guideline
- [x] Cross-reference research document

### Phase 2: Validation & Testing
**Complexity**: 2/10 | **Priority**: High

- [x] Validate markdown syntax is correct
- [x] Ensure backward compatibility (no behavior changes)
- [x] Verify instructions are clear and unambiguous
- [x] Test by reading instructions aloud (clarity check)
- [x] Peer review instructions for gaps

### Phase 3: Documentation Updates
**Complexity**: 1/10 | **Priority**: Medium

- [x] Update README.md to reference enhanced instructions
- [x] Update CHANGELOG.md with enhancement note
- [x] Link to research document in CHANGELOG

## Relevant Files

### Existing Files
- `commands/create-pr.md` - **Main file to enhance** (lines 14-28)
  - Currently has: title format, body structure, run steps, report format
  - Needs: explicit DO NOT rules, detailed guidance, examples

- `research/pr-description-best-practices.md` - **Source of truth for best practices**
  - Contains: industry standards, common mistakes, recommendations

- `README.md` - **Document reference**
  - Currently mentions: "Generate well-formatted pull requests from GitHub issues"
  - May need: brief mention of SDLC alignment

- `CHANGELOG.md` - **Record of enhancements**
  - Needs: Note about instruction enhancements

- `.claude-plugin/plugin.json` - **Plugin manifest**
  - No changes needed; references `commands/create-pr.md` by path

### New Files
None

### Test Files
None - no code changes, no tests needed

## Testing Strategy

### Unit Tests
Not applicable - instruction-only changes

### Integration Tests
Manual verification:
1. Read enhanced instructions clearly
2. Verify they don't break existing PR workflows
3. Verify they correctly forbid unwanted sections

### E2E Tests
Manual test case:
1. **Test Case**: Generate PR with enhanced instructions
   - Run `/create-pr <issue_number>` on a branch
   - Verify generated PR description follows new guidance
   - Confirm no redundant commits section
   - Confirm Testing Notes only if non-obvious

### Manual Test Cases

1. **Test Case: PR with Simple Change**
   - Steps:
     1. Create feature branch with minor change
     2. Run `/create-pr <issue>`
     3. Examine generated PR description
   - Expected: Clean summary without commits list or generic testing notes

2. **Test Case: PR with Complex Change**
   - Steps:
     1. Create feature branch with architectural change
     2. Run `/create-pr <issue>`
     3. Examine generated PR description
   - Expected: Includes Review Focus with file order and architecture decisions

3. **Test Case: PR with Non-Obvious Testing**
   - Steps:
     1. Create branch with change requiring manual UAT
     2. Run `/create-pr <issue>`
     3. Examine Testing Notes section
   - Expected: Includes manual scenario, not generic "Testing: Passed"

## Risk Assessment

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Instructions become too verbose | Low | Medium | Keep examples concise; link to research for deep context |
| Ambiguity in "DO NOT" rules | Low | Medium | Provide specific examples of violations |
| Backward compatibility issues | Very Low | High | Ensure all existing structures remain valid |

### Business Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Users resist new guidelines | Very Low | Low | Explain rationale; reference industry best practices |
| Time spent on PRs increases | Very Low | Low | Better instructions = faster review = net time savings |

### Mitigation Strategy
- Provide clear, concrete examples for each guideline
- Test on real branches before deployment
- Reference authoritative sources in instructions
- Link research document for deeper understanding

## Rollback Strategy

### Rollback Steps
1. Revert `commands/create-pr.md` to previous version
2. Revert `README.md` and `CHANGELOG.md` if updated
3. Restart Claude Code to reload plugins

### Rollback Conditions
- If new instructions cause more PRs to fail review
- If instructions are unclear to users
- If backward compatibility is broken

## Validation Commands

```bash
# Validate plugin structure
bun run validate

# Check markdown syntax of enhanced file
cat commands/create-pr.md | grep -c "DO NOT"  # Should show > 0

# Manual verification
# 1. Read enhanced instructions aloud for clarity
# 2. Generate sample PR to verify behavior
# 3. Check for redundant sections
```

## Acceptance Criteria

Define clear, testable success criteria:

- [ ] All explicit "DO NOT" rules added with examples
- [ ] Review Focus guidance includes file order and architectural decisions
- [ ] Testing Notes guidance includes non-obvious examples
- [ ] Instructions remain clear and scannable
- [ ] No breaking changes to command behavior
- [ ] Backward compatible with existing workflows
- [ ] Plugin validation passes
- [ ] CHANGELOG updated with enhancement note
- [ ] Generated PRs no longer include redundant commits sections
- [ ] Generated PRs have valuable Review Focus content

## Dependencies

### New Dependencies
None - instruction-only changes

### Dependency Updates
None

## Notes & Context

### Additional Context
This enhancement directly addresses user complaints about LLM-generated PRs including unnecessary information. By making the instructions more explicit about what NOT to include, we improve PR quality without changing command behavior.

The research document (`research/pr-description-best-practices.md`) provides authoritative backing for all recommendations and can be referenced in the CHANGELOG.

### Assumptions
- Current command behavior is correct
- Users will follow clearer instructions
- No additional CI/CD validation is needed immediately

### Constraints
- Must maintain backward compatibility
- Should not increase command complexity
- Instructions should remain in `create-pr.md` (not separate file)

### Related Tasks/Issues
- Research: `research/pr-description-best-practices.md` (completed)
- Future: PR description validation in CI
- Future: Configuration for custom PR templates

### References
- Research Document: `research/pr-description-best-practices.md`
- GitHub Best Practices: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/getting-started/best-practices-for-pull-requests
- GitHub Blog: https://github.blog/2015-01-21-how-to-write-the-perfect-pull-request/
- FreeCodeCamp Guide: https://www.freecodecamp.org/news/how-to-write-a-pull-request-description/

### Open Questions
- [ ] Should we add a maximum character limit for PR descriptions?
- [ ] Should the implementation plan link be required or optional?
- [ ] Do we need to reference the research document directly in instructions?
