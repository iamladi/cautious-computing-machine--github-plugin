---
description: Comprehensive Pull Request Review
---

# Pull Request Review

## Role

Conduct a thorough review of a GitHub pull request: understand the intent, assess the diff against that intent, and surface issues the PR author and maintainer should know about. Output is an organized report, not a merge decision — leave the approve/request-changes call to humans.

## Priorities

Finding coverage (everything worth surfacing gets surfaced) > Evidence (every finding cites `file:line`) > Actionability (fixes suggested, not just problems named) > Brevity

## Scope

`$ARGUMENTS` is a PR identifier: a number (`123`), a URL (`https://github.com/owner/repo/pull/456`), or `owner/repo#number`.

## Routing

Standard flow is the default — one reviewer covers every concern sequentially. Pass `--swarm` for a team of four specialized reviewers (security, performance, tests, architecture) working in parallel; that's the right choice for large diffs (>30 files) or PRs where one dimension dominates (security-critical changes, perf-sensitive code paths). Strip `--swarm` from the args; the remainder is the PR identifier.

If swarm mode is requested but `TeamCreate` isn't available:
```
Swarm mode requires agent teams. Set CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 in settings.json or environment.
Falling back to standard workflow.
```
Then continue with the PR identifier already parsed.

## PR context (both modes)

Fetch everything the review will touch before doing any analysis — partial context produces partial findings:

```bash
# metadata
gh pr view {PR_NUMBER} --json title,body,state,author,headRefName,baseRefName,url,commits,reviews,comments,labels,milestone

# diff and discussion
gh pr diff {PR_NUMBER}
gh pr view {PR_NUMBER} --comments
gh pr checks {PR_NUMBER}

# local checkout for file reads
gh pr checkout {PR_NUMBER}
BASE_BRANCH=$(gh pr view {PR_NUMBER} --json baseRefName -q .baseRefName)
```

Before reviewing, read the project conventions — they shape what "reasonable" looks like in this codebase:
- `CONTRIBUTING.md` — review guidelines.
- `CLAUDE.md` — project-specific instructions.
- `.github/PULL_REQUEST_TEMPLATE.md` — required PR content.
- CI config — what's auto-checked vs what the review needs to catch.

---

## Standard workflow

### Analyze

Compare against the base branch and walk every change:

```bash
git diff $BASE_BRANCH...HEAD --stat
git diff $BASE_BRANCH...HEAD
git log $BASE_BRANCH..HEAD --no-merges
```

For every changed file, understand the change on its own terms. New files — what's their purpose? Deleted files — why were they removed? Modified files — what semantic change was made? Test files — does the new code have test coverage commensurate with its risk? Documentation — did the docs that describe this code get updated too?

Also fold in the discussion: unresolved review threads, CI status, linked issues. Context from a conversation often explains why a choice was made that looks wrong at the file level.

### Find stage

Collect every issue worth surfacing, no self-censoring. 4.7 will suppress "minor" findings if the prompt says "only surface important issues" — don't let it. For each finding, record:

- **Severity**: `blocker` | `major` | `minor` | `nit`
- **Confidence**: `high` | `medium` | `low`
- **Location**: `file:line` (or `file` for global concerns)
- **Category**: `correctness` / `security` / `performance` / `tests` / `docs` / `style` / `architecture`
- **One-line reasoning** — what makes this a finding, not a preference

Cover every axis. If you're only finding issues in one dimension, you're not looking hard enough at the others.

Severity calibration (apply to every finding, not just the obvious ones):
- **Blocker** — merge would introduce a correctness bug, security hole, or break a deployed contract.
- **Major** — likely to cause noticeable problems in production or maintenance, but merge isn't unsafe.
- **Minor** — smells, inconsistencies, or small cleanup opportunities; fine to defer.
- **Nit** — style, wording, or personal preference with negligible impact.

Confidence calibration:
- **High** — you traced the issue through the code and can cite the exact mechanism.
- **Medium** — the issue is probable based on pattern, but you didn't verify end-to-end.
- **Low** — suspicious but you'd want someone to double-check.

### Filter stage

From the find-stage list, surface to the report:
- Every `blocker` (any confidence).
- Every `major` with confidence ≥ medium.
- `minor` findings grouped as a single section, one line each.
- `nit` findings as an optional trailing group, one line each.

Drop low-confidence `major` or below unless two or more independently suggest the same area — in which case surface it and flag the uncertainty.

### Report

Structure the output for the PR author to act on, not just read. Use `file:line` everywhere a location exists.

```markdown
# PR Review: #{number} — {title}

## Summary
- **Purpose**: {one-line description of what the PR does}
- **Scope**: {feature / bugfix / refactor / docs / chore / ...}
- **Risk**: {low / medium / high — and why}
- **Overall**: {thumbnail verdict — "looks solid", "ready pending fixes", "needs discussion", etc. — this is input to human reviewers, not a merge decision}

## Findings

### Blockers
- **[correctness, high conf]** `src/auth.ts:42` — {description + execution path}. Fix: {suggested change}.

### Major
- **[security, medium conf]** `src/api/handler.ts:118–140` — {description}. Fix: {suggested change}.

### Minor
- `src/util.ts:8` — variable name suggests the wrong invariant.
- `tests/form.spec.ts:55` — assertion relies on internal shape; could be brittle.

### Nits
- `README.md:L24` — typo "recieve".

## Discussion & CI

- Unresolved threads: {summary or "none"}
- CI: {passing / N failing / pending}
- Linked issues: {list}

## Positive notes

- {genuine praise for well-done parts — not boilerplate}

## Suggested follow-up session

Offer: deep-dive into a specific file, run tests, search for similar patterns elsewhere, or draft a GitHub review comment from the findings above.
```

---

## Swarm workflow

Parallelize by dimension when the PR is big or multi-faceted. Four specialized reviewers run concurrently, each covering one concern; findings get consolidated with the same find/filter discipline.

### Team setup

Create the team with `TeamCreate`: name `review-pr-{number}-{YYYYMMDD-HHMMSS}`, description `PR Review: {title}`. Register four shared tasks via `TaskCreate`: Security, Performance, Test Coverage, Architecture.

### Context for teammates

Teammates can't see this conversation. Every spawn prompt embeds literally:

- PR title, author, base and head branches.
- Diff summary: `git diff $BASE_BRANCH...HEAD --stat`.
- Changed file list.
- PR description/body.
- For PRs ≤50 files: include the full changed file contents in the prompt. For PRs >50 files: include the file list only and tell the teammate to `Read` what they need.

### Teammate dispatch

Spawn all four via `Task` with `team_name` and `subagent_type: "general-purpose"`. The prompts share most of their structure; the specialization is in the role, judgment criteria, and cross-concern triggers:

<example name="Specialized reviewer prompt template">
You are conducting a {Security | Performance | Test Coverage | Architecture} review as part of a PR review team.

PR TITLE: {literal}
PR AUTHOR: {literal}
BASE BRANCH: {literal}
HEAD BRANCH: {literal}

DIFF SUMMARY:
{literal stat output}

CHANGED FILES:
{literal list}

{If ≤50 files:}
CHANGED FILE CONTENTS:
{literal contents}

{If >50 files:}
Note: >50 changed files. Use `Read` to examine files relevant to your concern.

PR DESCRIPTION:
{literal body}

YOUR ROLE — {role-specific sentence}:
- Security: find vulnerabilities and risks an attacker could exploit or that expose data.
- Performance: find changes that will degrade response times, increase resource use, or create scale bottlenecks.
- Test Coverage: assess whether the suite adequately covers the changes and would catch regressions.
- Architecture: evaluate design patterns, separation of concerns, and fit with codebase conventions.

SEVERITY — calibrate to your dimension:
- Critical: {role-specific example — e.g., "unsanitized user input reaching a SQL query" / "N+1 in hot path" / "new API endpoint with no tests" / "breaking change to public API"}.
- High: {role-specific example — e.g., "missing auth on internal endpoint" / "missing index on growing table" / "error paths untested" / "significant deviation from codebase conventions"}.
- Medium: {role-specific example — e.g., "verbose error messages leaking internals" / "unnecessary allocations in loops" / "shallow tests that only cover happy path" / "logic in wrong layer"}.
- Low: {role-specific example — best practice violations with minimal real impact}.

Apply find/filter discipline: note *every* issue you see at every severity, then surface all high+critical and a representative sample of medium/low. 4.7 otherwise suppresses low-severity findings the team lead actually wants.

CROSS-CONCERN FINDINGS:
When a finding spans concerns (e.g., a security issue also has perf implications), share via `SendMessage`:
"{Your role}: {issue} at {file}:{line} also has {other concern} implications — {explanation}"

CONSTRAINTS (parallel-team context):
- Read-only. Don't modify the codebase or run build/test commands — other reviewers are working concurrently.
- Communicate via `SendMessage` only; `AskUserQuestion` isn't available in team context.

COMPLETION:
1. `TaskUpdate` your shared task with findings.
2. `SendMessage` `REVIEW COMPLETE`.
3. Wait for `shutdown_request`.

For each finding: `file:line`, severity, description, recommendation. Match depth to severity — critical findings deserve a traced execution path; nits deserve one line.
</example>

### Convergence

Wait for all four `REVIEW COMPLETE` messages, up to 10 minutes per teammate from spawn. On teammate timeout, consolidate with whatever arrived and note which dimension is missing — a partial review is more useful than no review. If a teammate is stuck (repeated messages, no progress): note the failure, respawn with tighter scope, or absorb that dimension into the lead — pick on criticality.

### Consolidation

Same find/filter discipline as standard workflow, but with reviewer attribution: `[Security]`, `[Performance]`, `[Test Coverage]`, `[Architecture]`, or `[Consensus]` when two or more reviewers flagged the same area. Cross-cutting findings get their own callout:

> `[Consensus]` The dependency update at `package.json:42` raises both security concerns (known CVE) and performance concerns (increased bundle size).

Output format is identical to standard — the user shouldn't need to know which mode ran.

### Cleanup invariant

**The team must be deleted before the command returns, regardless of whether consolidation succeeded.** Skipping leaks team slots and orphans the shared task list.

1. `SendMessage` `type: "shutdown_request"` to each teammate.
2. Wait briefly for shutdown confirmations.
3. `TeamDelete`.

If cleanup itself errors, tell the user `"Team cleanup incomplete. You may need to check for lingering team resources."` and continue to output.

---

## Interactive follow-up

After the report, offer the user:
- Deep-dive into a specific file.
- Run the test suite (if the repo has a clear test command) to verify the PR claims.
- Search the codebase for similar patterns elsewhere.
- Draft a GitHub review comment from the findings.

These are optional — the primary output is the report.

## Error responses

Keep the user unblocked:

```
PR not found          — Verify PR number/URL; `gh pr list` to browse.
Not authenticated     — Run: gh auth login
Uncommitted changes   — Stash or commit before `gh pr checkout`.
Merge conflicts       — Note conflicts; suggest resolving before review.
Network issue         — Suggest retry or manual `gh` command.
```

## Safety

Never auto-commit, auto-push, or run destructive git operations. This command is pure analysis — changes come from the PR author following the findings, not from the reviewer.

## Usage

```bash
/review-pr 123                                              # current repo
/review-pr https://github.com/owner/repo/pull/456           # any repo
/review-pr owner/repo#789                                   # explicit repo
/review-pr 123 --swarm                                      # parallel specialized reviewers
```
