---
description: Create GitHub Pull Request
---

# Create Pull Request

Based on the `Instructions` below, take the `Variables` follow the `Run` section to create a pull request. Then follow the `Report` section to report the results of your work.

## Variables

issue: $1
plan_file: $2

## Instructions

### PR Title Format
Generate a pull request title in the format: `<issue_type>: #<issue_number> - <issue_title>`
- Extract issue number, type, and title from the issue JSON
  - Use `gh issue view <issue_number> --json number,title,body` to fetch issue details
- Examples of PR titles:
  - `feat: #123 - Add user authentication`
  - `bug: #456 - Fix login validation error`
  - `chore: #789 - Update dependencies`

### PR Body Structure

The PR body should communicate the **why** and **what**, not duplicate information already visible in the PR interface. Include these sections in this order:

#### 1. Summary (Purpose & Context)
**Explain the "why"** - business reason, goals, and impact of the change.

- Describe the problem being solved or feature being added
- Explain the business or technical rationale (don't assume reviewer context)
- Keep it concise but complete (1-3 sentences is often sufficient)
- Example: "Adds email verification flow required for HIPAA compliance (see #456). Users receive verification emails during signup and must verify before accessing protected data."

#### 2. Review Focus
**Highlight what reviewers should pay attention to.** This is where you add genuine value beyond code review.

Include:
- **Files to review in priority order** (for complex PRs): "Start with `auth.ts` for the core logic, then `utils.ts` for the helper functions"
- **Architectural decisions made**: "Chose async/await over promises for clarity; added caching layer to reduce DB queries"
- **Potential gotchas or edge cases**: "Error handling for expired tokens; special case for admin users who skip verification"
- **Breaking changes** (if any): "Endpoint `/api/auth/verify` requires email parameter; old `/api/auth/check` is deprecated"

Do NOT include:
- Generic placeholder text like "Please review carefully" or "Look for bugs"
- Line-by-line code explanations (those belong in code comments)
- Generic section headers without substance

Example Review Focus:
```
## Review Focus

**Files to review (in order):**
1. `src/auth.ts` - Core verification logic
2. `src/email.ts` - Email delivery and retry logic
3. `tests/auth.test.ts` - Edge cases for expired/invalid tokens

**Key decisions:**
- Used async/await pattern for clarity (vs Promise chains)
- Implemented retry logic for transient email failures
- Added grace period for token expiration (5 minutes) for slow networks

**Edge cases to verify:**
- Expired tokens during verification flow
- Admin users who can skip verification
- Race conditions if user attempts multiple verifications
```

#### 3. References
- **Reference to the issue**: `Closes #<issue_number>` (automatic PR-to-issue linking)
- **Link to implementation plan**: Extract from plan file parameter
  - If plan file provided: Include "See `plans/filename.md` for full implementation design"
  - Extract research links from plan frontmatter if available
- **Related PRs or documentation**: Link to dependent PRs or relevant docs

**Example:**
```
## Plan & Design
See [`plans/add-oauth2-auth.md`](../plans/add-oauth2-auth.md) for full implementation design and validation criteria.

Related research:
- [`research/auth-flow.md`](../research/auth-flow.md)
```

#### 4. Testing Notes (Default: Omit)
**Omit this section entirely unless** you need to:
- Request a specific manual action from the reviewer (e.g., test in staging, verify multi-browser behavior)
- Report manual verification you performed that CI cannot cover (e.g., "Verified on iOS Safari")

If all testing is automated (unit tests, integration tests, linting, type checks), do NOT include this section.

Example (only when needed):
```
## Testing Notes

- Verified email delivery in staging (SMTP logs confirm 3 test emails sent)
- Reviewer: please test token expiration in multi-tab scenario (open two browser windows, expire token in first, verify second detects it)
```

### What NOT to Include

These belong elsewhere and waste reviewer time:

- **❌ Separate "Commits" section**: Commit history is already visible in the PR's "Commits" tab. Summarize changes instead.
  - Bad: "1. Add email verification logic\n2. Update database schema\n3. Add tests"
  - Good: "Implements email verification with async/await pattern and database schema changes"

- **❌ Test coverage documentation**: CI/CD pipeline results are visible in PR checks. Trust the automation.
  - Bad: "Wrote 12 unit tests covering all edge cases. All tests passing."
  - Good: (Only mention if manual testing is needed beyond CI)

- **❌ Implementation details that belong in code**: Line-by-line code explanations, algorithm pseudocode, or logic flows.
  - Bad: "Uses a for loop to iterate through users and validates each email"
  - Good: (Add clarifying comments in the code itself)

- **❌ Unverified assumptions about context**: Assume reviewers need context provided.
  - Bad: "As discussed in the meeting yesterday"
  - Good: "As outlined in the requirements doc (see project wiki link)"

- **❌ Filler or boilerplate**: Empty sections or placeholder text.
  - Bad: "## Changes\nSee commits for details" or "## Testing\nCI checks passed"
  - Good: (Skip empty sections entirely)

### Scope Guideline
Keep PR descriptions concise—they should be **shorter than the actual code changes**, not longer. If your description is longer than the code, it likely contains information that belongs elsewhere (comments, docs, design docs).

## Run

1. **Fetch Issue details**: `gh issue view <issue_number> --json number,title,body,labels`
2. **Read plan file** (if provided):
   - Parse YAML frontmatter to extract `type`, `research` fields
   - Extract key sections: Overview, Implementation Plan phases
3. **Generate PR title**: `<type>: #<issue_number> - <issue_title>`
   - Use type from plan frontmatter (defaults to issue labels if not in plan)
4. **Generate PR body**:
   - Summary: From plan's Overview section
   - Review Focus: Key decisions from Implementation Plan
   - Plan & Design: Link to plan file + research files
   - References: `Closes #<issue_number>` + plan link
   - Testing Notes: Only if manual reviewer action needed or manual verification to report (default: omit)
5. Run `git push -u origin <branch_name>` to push the branch
6. Run `gh pr create --title "<pr_title>" --body "<pr_body>" --base main` to create the PR
7. Capture the PR URL from the output

## Report

Return ONLY the PR URL that was created (no other text)
