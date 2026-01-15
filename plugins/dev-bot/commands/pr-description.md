---
allowed-tools: Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh api:*), Bash(git log:*), Bash(git branch:*)
description: Generate or update PR title and description
---

Generate a comprehensive PR title and description with automated analysis.

Arguments:
- `PR_NUMBER` (optional): The PR number to update. If not provided, uses the current branch's PR.

## Steps

1. **Fetch PR and branch info**: Get PR metadata, diff, and branch name using `gh pr view` and `gh pr diff`. Extract the Linear ticket reference from the branch name (e.g., `lle-1234-feature-name` → `LLE-1234`).

2. **Preserve existing user description**: If updating an existing PR, extract and preserve the content from the "User Description" section. This content should be kept verbatim in the updated description.

3. **Analyse changes in parallel**: Launch multiple Opus agents in parallel to analyse different aspects:

   **Agent 1 (Summary Writer)**: Write a concise technical summary of the changes:
   - What problem does this PR solve?
   - What approach was taken?
   - Key technical decisions made
   - Any breaking changes or migration requirements

   **Agent 2 (File Analyser)**: Categorise all changed files and write brief descriptions:
   - Group files by category (Enhancement, Bug fix, Configuration, Tests, Documentation, etc.)
   - Write a one-line description of what changed in each file
   - Calculate line changes (+/-) for each file

   **Agent 3 (Risk Assessor)**: Evaluate the PR for potential issues:
   - Identify any obvious bugs or issues
   - Check for missing error handling, edge cases
   - Look for security concerns
   - Assign a confidence score (1-5) with explanation

4. **Generate PR title**: Create a title in the format `[{LINEAR_TICKET}] - {Concise description}`:
   - Extract Linear ticket from branch name (e.g., `feature/lle-1234-add-auth` → `LLE-1234`)
   - Write a concise description (50 chars max after ticket)
   - Use imperative mood (e.g., "Add", "Fix", "Update", "Remove")

5. **Assemble description**: Combine all sections using the template below. If there was an existing user description, preserve it in the User Description section.

6. **Update PR**: Use `gh pr edit` to update the title and body.

---

## Template

```markdown
Closes #{LINEAR_TICKET_NUMBER}

## User Description

{preserved_user_description_or_empty}

---

## Summary

{ai_generated_summary}

## Confidence Score: {score}/5

{confidence_explanation}

{specific_concerns_if_any}

<details>
<summary><h3>File Walkthrough</h3></summary>

<table>
<thead><tr><th></th><th align="left">Relevant files</th></tr></thead>
<tbody>
<tr>
<td><strong>{Category1}</strong></td>
<td>
<details><summary>{n} files</summary>
<table>
<tr>
  <td><strong>{filename}</strong><dd><code>{brief_description}</code></dd></td>
  <td><a href="{file_diff_url}">{+lines/-lines}</a></td>
</tr>
</table>
</details>
</td>
</tr>
</tbody>
</table>

</details>
```

---

## File Categories

Use these categories to group files:

| Category | Use for |
|----------|---------|
| **Enhancement** | New features, improvements to existing functionality |
| **Bug fix** | Fixes for bugs or incorrect behaviour |
| **Refactoring** | Code restructuring without behaviour change |
| **Configuration** | Config files, schemas, migrations, environment |
| **Tests** | Test files, test utilities, fixtures |
| **Documentation** | README, docs, comments |
| **Dependencies** | Package.json, lock files, dependency updates |
| **Infrastructure** | CI/CD, Docker, deployment configs |

---

## Confidence Score Guide

| Score | Meaning |
|-------|---------|
| 5 | No concerns - straightforward, well-tested changes |
| 4 | Minor concerns - small improvements possible |
| 3 | Moderate concerns - some areas need attention |
| 2 | Significant concerns - potential bugs or issues identified |
| 1 | Critical concerns - likely bugs that will cause failures |

When score is 3 or below, include specific concerns with file references.

---

## Notes

- Always preserve existing user descriptions when updating
- Linear ticket is extracted from branch name patterns: `lle-1234`, `feature/lle-1234-name`, `lle-1234-feature-name`
- If no Linear ticket found, omit the `Closes #` line and use descriptive title without ticket prefix
- File diff URLs should link to the specific file in the PR diff view
- Keep summary focused on technical changes, not implementation details
- Confidence score should be honest - low scores help catch issues before merge
