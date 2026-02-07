---
title: "Fix create-pr leaking Commits and Test Plan sections"
type: Enhancement
issue: 10
research:
  - research/research-sdlc-submit-pr-description.md
  - research/pr-description-best-practices.md
status: Ready for Implementation
reviewed: true
reviewers: ["codex", "gemini"]
created: 2026-02-07
---

# PRD: Fix create-pr leaking Commits and Test Plan sections

## Metadata
- **Type**: Enhancement
- **Priority**: High
- **Severity**: Major (PR quality issue affecting reviewers)
- **Estimated Complexity**: 2/10
- **Created**: 2026-02-07
- **Status**: Ready for Implementation

## Overview

### Problem Statement
The `/github:create-pr` command generates PR descriptions that intermittently include a "Commits" section and a "Test plan" section despite explicit exclusion rules in `create-pr.md` (lines 110-132).

**Root cause**: Two structural issues in the prompt that undermine the exclusion rules:

1. **`git log` primes the model with commit data** — Step 5 in the Run section (`git log origin/main..HEAD --oneline`) loads commit messages into context right before body generation. Despite the "DO NOT include commits" rule, the model sees commit data and sometimes formats it into the body.

2. **Testing Notes section is too permissive** — The current "Optional - Only if Non-Obvious" framing leaves too much room for interpretation. When the model ran tests during the session or has test output in context, it rationalizes including a test checklist. The section needs a stronger default: omit unless there's a specific manual action the reviewer must take.

A secondary issue exists in `workflows-plugin/commands/phase-submit.md` (lines 40-44) which explicitly asks for "Test plan" in the PR body, contradicting `create-pr.md`. This affects the workflows path, not the SDLC submit path directly, but should be fixed for consistency.

### Goals & Objectives

1. Eliminate commits leaking into PR body by removing `git log` from pre-generation steps (it serves no purpose — the model already sees the diff)
2. Make Testing Notes default to omission with an explicit trigger condition
3. Align `phase-submit.md` with `create-pr.md` so it doesn't contradict the exclusion rules

### Success Metrics
- **Primary Metric**: Generated PRs never contain a "Commits" section or a generic "Test plan" section
- **Quality Gates**:
  - `create-pr.md` no longer runs `git log` before body generation
  - Testing Notes section clearly states the default is omission
  - `phase-submit.md` no longer lists "Test plan" as a PR body requirement

## User Stories

### Story 1: No redundant commit listing
- **As a**: Code reviewer
- **I want**: PR descriptions that don't list commits
- **So that**: I use the PR's native Commits tab for commit history
- **Acceptance Criteria**:
  - [ ] `git log` step removed from Run section
  - [ ] No path for commit data to enter body generation context

### Story 2: No generic test plan
- **As a**: Code reviewer
- **I want**: PR descriptions that omit testing info when CI covers everything
- **So that**: I trust CI and only see testing notes when manual action is genuinely needed
- **Acceptance Criteria**:
  - [ ] Testing Notes section defaults to "omit this section entirely"
  - [ ] Only included when reviewer must take a specific manual action (e.g., test in staging, multi-browser check)

## Requirements

### Functional Requirements

1. **FR-1**: Remove `git log` from Run steps in `create-pr.md`
   - Details: Delete step 5 (`git log origin/main..HEAD --oneline`). Renumber subsequent steps.
   - Priority: Must Have

2. **FR-2**: Rewrite Testing Notes section to default-omit
   - Details: Change from "Optional - Only if Non-Obvious" to a clear default-omit with explicit trigger. The section title should signal omission: "Testing Notes (Default: Omit)"
   - Include two valid triggers: (a) reviewer must take a specific manual action, (b) author reporting manual verification results (e.g., "Verified on iOS Safari")
   - Priority: Must Have
   <!-- Addressed: Softened per Codex+Gemini consensus — allows reporting performed manual verification, not just requesting it -->

3. **FR-3**: Update `phase-submit.md` to remove "Test plan" from PR body list
   - Details: Remove "Test plan" bullet. Keep "List of implemented plans" (workflows may depend on it). Add note that body structure follows `/github:create-pr`.
   - Priority: Should Have

### Non-Functional Requirements

1. **NFR-1**: Backward Compatibility
   - Requirement: No changes to command invocation, arguments, or output format
   - Target: 100% compatible
   - Measurement: Plugin validation passes

## Scope

### In Scope
- Edit `github-plugin/commands/create-pr.md` — remove git log step, strengthen Testing Notes
- Edit `workflows-plugin/commands/phase-submit.md` — remove contradicting "Test plan" instruction
- Update `github-plugin/CHANGELOG.md`

### Out of Scope
- Changes to PR title format
- Changes to other sections (Summary, Review Focus, References)
- Changes to `sdlc-plugin/commands/submit.md` (already correct — delegates cleanly)
- Adding eval tests (github-plugin has no eval infrastructure)

### Future Considerations
- Eval tests for PR body quality (structural assertions for absence of "Commits" heading)

## Impact Analysis

### Affected Areas
- `github-plugin/commands/create-pr.md` — Run section and Testing Notes
- `workflows-plugin/commands/phase-submit.md` — PR body description

### Users Affected
- All users of `/github:create-pr` and `/sdlc:submit`
- All users of `/workflows:*` pipeline

### System Impact
- **Performance**: None
- **Security**: None
- **Data Integrity**: None

### Dependencies
- None

### Breaking Changes
- [x] **None** — instruction-only changes, same command interface

## Root Cause Analysis

1. **Why** do commits appear in the PR body?
   - Step 5 runs `git log origin/main..HEAD --oneline` which loads commit data into context
2. **Why** does the model include it despite exclusion rules?
   - LLMs tend to use available data. The exclusion rule is in a different section ("What NOT to Include") than the Run steps, creating a distance between "here's data" and "don't use it"
3. **Why** does the test plan appear?
   - The Testing Notes section says "Optional" which the model interprets as "include if you have anything to say." The model always has something to say about testing.
4. **Why** does phase-submit contradict create-pr?
   - It was written before the exclusion rules were added to create-pr.md (v1.2.1)
5. **Root Cause**: The `git log` step feeds data the model shouldn't format, and the Testing Notes framing is too permissive

## Solution Design

### Approach

Three targeted edits:

**Edit 1: Remove `git log` from `create-pr.md` Run section (line 151)**
Delete step 5 and renumber steps 6-8 to 5-7. The model doesn't need commit history to generate the body — it has the issue, the plan, and the diff.

**Edit 2: Rewrite Testing Notes in `create-pr.md` (lines 87-108)**
Change the section heading and opening line to make omission the strong default:
- Heading: `#### 4. Testing Notes (Default: Omit)`
- Opening: `**Omit this section entirely unless** you need to: (a) request specific manual action from the reviewer, or (b) report manual verification you performed that CI cannot cover.`
- Add: `If all testing is automated (unit tests, integration tests, linting, type checks), do NOT include this section.`
<!-- Addressed: Gemini+Codex consensus — allows reporting performed verification results, not just requesting actions -->

**Edit 3: Update `phase-submit.md` (lines 38-45)**
Remove only "Test plan" from the bullet list. Keep "List of implemented plans" (workflows pipeline may depend on it for multi-plan PRs). Add note that PR body structure follows `/github:create-pr` instructions.
<!-- Addressed: Codex concern — "List of implemented plans" may be required by workflows pipeline, only remove the contradicting "Test plan" item -->

### Alternatives Considered

1. **Alternative 1**: Move `git log` to after `gh pr create` (for informational logging only)
   - Pros: Still shows commits in session output
   - Cons: Unnecessary — the model doesn't need it at all
   - Why rejected: Simpler to remove entirely

2. **Alternative 2**: Add stronger "NEVER" language to exclusion rules without removing git log
   - Pros: Less structural change
   - Cons: Doesn't fix the root cause — data still enters context
   - Why rejected: Removing the data source is more reliable than hoping the model ignores it

## Implementation Plan

### Phase 1: Edit `create-pr.md`
**Complexity**: 2/10 | **Priority**: High

- [ ] Remove step 5 (`git log origin/main..HEAD --oneline`) from Run section
- [ ] Renumber steps 6-8 → 5-7
- [ ] Search for and update any text references to step numbers or `git log` output within the file
- [ ] Rewrite Testing Notes section (lines 87-108) heading and opening to default-omit
- [ ] Add two valid triggers: (a) requesting manual reviewer action, (b) reporting performed manual verification
<!-- Addressed: Gemini — broken step number reference risk -->

### Phase 2: Edit `phase-submit.md`
**Complexity**: 1/10 | **Priority**: Medium

- [ ] Remove "Test plan" from the PR body bullet list (lines 40-44)
- [ ] Keep "List of implemented plans" item (workflows pipeline may depend on it)
- [ ] Add note that PR body follows `/github:create-pr` body structure

### Phase 3: Update CHANGELOGs
**Complexity**: 1/10 | **Priority**: Low

- [ ] Add entry to `github-plugin/CHANGELOG.md` for this fix
- [ ] Add entry to `workflows-plugin/CHANGELOG.md` if it exists (for phase-submit change)
<!-- Addressed: Codex — workflows plugin change needs its own release note -->

### Phase 4: Validation
**Complexity**: 1/10 | **Priority**: High

- [ ] Run `bun run validate` in github-plugin
- [ ] Run `bun run validate` in workflows-plugin (if exists)
- [ ] Manual review: read `create-pr.md` end-to-end for coherence

## Relevant Files

### Existing Files
- `github-plugin/commands/create-pr.md` — Main file: remove git log step, rewrite Testing Notes
- `workflows-plugin/commands/phase-submit.md` — Remove contradicting "Test plan" instruction
- `github-plugin/CHANGELOG.md` — Add entry
- `github-plugin/README.md` — No changes needed (already correctly documents behavior)

### New Files
None

### Test Files
None (github-plugin has no eval infrastructure)

## Testing Strategy

### Manual Test Cases

1. **Test Case: Read through create-pr.md**
   - Steps: Read the file end-to-end
   - Expected: No `git log` in Run steps. Testing Notes clearly defaults to omission. No broken step number references.

2. **Test Case: Read through phase-submit.md**
   - Steps: Read the file end-to-end
   - Expected: No "Test plan" in PR body requirements. "List of implemented plans" still present.

3. **Test Case: Plugin validation**
   - Steps: Run `bun run validate` in both plugins
   - Expected: Pass with zero errors

4. **Test Case: Functional smoke test**
   - Steps: Execute `/github:create-pr` on a real branch with changes
   - Expected: Generated PR body has no "Commits" section and no generic "Test plan" section. Command executes without error.
   <!-- Addressed: Consensus (Codex+Gemini) — functional verification, not just static checks -->

## Risk Assessment

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Model still includes commits from session context (not git log) | Low | Low | The exclusion rules remain; removing git log just eliminates the strongest trigger |
| Testing Notes omitted when manual testing IS needed | Very Low | Medium | Two triggers: requesting manual action OR reporting performed verification |
| Large diff + high-level plan = weaker summary without git log context | Low | Low | Accept trade-off — issue + plan + diff provide sufficient context for summary |

### Mitigation Strategy
- The exclusion rules in "What NOT to Include" section remain as a safety net
- The Testing Notes section still allows inclusion with a clear trigger

## Rollback Strategy

### Rollback Steps
1. `git revert <commit>` in `github-plugin` repo
2. `git revert <commit>` in `workflows-plugin` repo (if phase-submit was changed)
3. Restart Claude Code to reload plugins
<!-- Addressed: Codex — multi-repo rollback needs per-repo revert -->

### Rollback Conditions
- If PRs consistently omit Testing Notes that should be included

## Validation Commands

```bash
# Validate github-plugin structure
cd github-plugin && bun run validate

# Validate workflows-plugin structure (if exists)
cd workflows-plugin && bun run validate

# Verify git log removed from create-pr
grep -c "git log" github-plugin/commands/create-pr.md  # Should be 0

# Verify Test plan removed from phase-submit
grep -c "Test plan" workflows-plugin/commands/phase-submit.md  # Should be 0
```

## Acceptance Criteria

- [ ] `create-pr.md` Run section no longer includes `git log` step
- [ ] Testing Notes heading and opening line default to omission
- [ ] `phase-submit.md` no longer lists "Test plan" in PR body requirements
- [ ] Plugin validation passes for both plugins
- [ ] No breaking changes to command invocation or output

## Dependencies

### New Dependencies
None

### Dependency Updates
None

## Notes & Context

### Additional Context
The exclusion rules in `create-pr.md` (lines 110-132) were added in v1.2.1 and are well-written. The issue isn't the rules themselves — it's that the Run section feeds commit data into context (undermining the "no commits" rule) and the Testing Notes section frames inclusion as the easy path (undermining the "no test coverage" rule).

### Assumptions
- Removing `git log` has no downstream effect — it was "informational" only
- The model generates better body content from issue + plan + diff than from commit messages

### Constraints
- Must maintain backward compatibility with existing invocation patterns
- Both plugins are in separate git repos

### Related Tasks/Issues
- Research: `research/research-sdlc-submit-pr-description.md`
- Previous research: `github-plugin/research/pr-description-best-practices.md`
- Previous plan (completed): `github-plugin/plans/enhance-create-pr-instructions.md`

### Open Questions
- [x] ~~Should `git log` be moved to after PR creation instead of removed?~~ No — removing entirely is simpler and the model doesn't need it. The diff + issue + plan provide sufficient context. Accepted trade-off per Gemini review.

## Blindspot Review

**Reviewers**: GPT-5.2-Codex (xhigh), Gemini 3 Pro
**Date**: 2026-02-07
**Plan Readiness**: Ready for Implementation

### Addressed Concerns
- [Consensus] Testing Notes too strict / over-suppression → Softened FR-2 and Edit 2 to allow reporting performed manual verification, not just requesting actions
- [Consensus] No functional verification → Added Test Case 4 (smoke test with real branch)
- [Gemini] Broken step number references after renumbering → Added explicit checklist item in Phase 1
- [Codex] Phase-submit removes potentially required content → Changed to only remove "Test plan", keep "List of implemented plans"
- [Codex] Workflows plugin change without release notes → Added workflows-plugin CHANGELOG to Phase 3
- [Codex] Rollback under-specified for multi-repo → Updated rollback steps to specify per-repo reverts

### Acknowledged but Deferred
- [Codex, High] Other commit-data sources not checked → The research document already confirmed `git log` is the only commit-injecting step in `create-pr.md`. Other plugins don't inject commits. Deferring broader scan.
- [Codex, Medium] Success metrics not directly validated → Addressed via Test Case 4 functional smoke test

### Dismissed
- [Gemini, Low] Context starvation for summaries without git log → Issue + plan + diff provide sufficient context. Commit messages are a subset of this information. Trade-off accepted and added to risk table.
