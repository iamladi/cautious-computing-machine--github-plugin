---
description: Create GitHub Issue from plan file
---

# Create Issue from Plan

Based on the plan file provided, create a GitHub Issue with the implementation phases as checkboxes. Then update the plan file with the Issue number.

## Variables

plan_file: $1

## Instructions

### 1. Parse Plan File

Read the plan file and extract:
- **Frontmatter**: `title`, `type`, `created`
- **Overview**: Problem statement, goals
- **Implementation Plan**: All phases with their tasks
- **Validation Commands**: From the Validation Commands section
- **Acceptance Criteria**: From the Acceptance Criteria section
- **Research links**: From frontmatter `research` field (if present)

### 2. Generate Issue Body

Structure the Issue body as:

```markdown
## Summary

[Extract from plan Overview > Problem Statement and Goals]

---

## Implementation Phases

### Phase 1: [Title from plan]
**Complexity**: [from plan] | **Priority**: [from plan]

- [ ] [Task 1]
- [ ] [Task 2]

### Phase 2: [Title from plan]
...

[Continue for all phases]

---

## Validation Commands

[Extract the bash commands from plan's Validation Commands section]

---

## Acceptance Criteria

- [ ] [From plan]
- [ ] [From plan]

---

## Related Research

[If research files listed in frontmatter, include links]

---

**Plan Document**: plans/[filename].md
**Issue Type**: [type from frontmatter]
**Created**: [from frontmatter]
```

### 3. Create the Issue

1. Determine issue type label from plan `type` field:
   - `Feature` → `type:feature`
   - `Bug` → `type:bug`
   - `Chore` → `type:chore`
   - `Refactor` → `type:refactor`
   - `Enhancement` → `type:enhancement`
   - `Documentation` → `type:documentation`

2. Create Issue via `gh issue create`:
   ```bash
   gh issue create \
     --title "[type]: [Plan Title]" \
     --body "[generated body above]" \
     --label "status:planning" \
     --label "[type_label]"
   ```

3. Capture Issue number from output (will be `#123`)

### 4. Update Plan File with Issue Number

1. Parse plan YAML frontmatter
2. Update `issue: null` → `issue: 123`
3. Update `status: Draft` → `status: In Progress`
4. Write updated plan back to file
5. Commit: `git add plans/[filename].md && git commit -m "chore: link plan to Issue #123"`

## Run

1. Validate plan file exists: `test -f "$plan_file"` or error
2. Read and parse plan file (extract sections above)
3. Generate Issue body using template
4. Create Issue: `gh issue create --title "..." --body "..." --label "..."`
5. Extract Issue number from output
6. Update plan frontmatter with Issue number
7. Commit update to plan file
8. Return: "Issue #123 created from plan. Updated plan frontmatter."

## Report

Return the Issue URL created:
- Format: `https://github.com/<owner>/<repo>/issues/<number>`
- Example: `https://github.com/iamladi/my-project/issues/123`
