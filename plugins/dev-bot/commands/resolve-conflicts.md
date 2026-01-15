---
allowed-tools: Bash(git:*), Bash(gh:*), Bash(pnpm type-check:*), Bash(pnpm lint:*)
description: Automatically resolve git merge conflicts on a PR branch and push
disable-model-invocation: false
---

Automatically resolve all merge conflicts on a PR branch, preserving features from both branches, then push.

Arguments:
- PR number or branch name (required)

## Steps

1. **Setup**: Checkout the PR branch with `gh pr checkout <pr-number>`, fetch origin main, then attempt `git merge origin/main --no-commit --no-ff`. If no conflicts, abort and report "No conflicts to resolve".

2. **Analyse both branches**: Get the merge base with `git merge-base HEAD origin/main` and list conflicted files with `git diff --name-only --diff-filter=U`. Launch 2 parallel Sonnet agents:
   - **Agent 1 (PR Branch)**: Run `git diff $MERGE_BASE..HEAD -- <conflicted-files>`. List every feature, function, import, prop, and hook this branch adds. Be exhaustive.
   - **Agent 2 (Main Branch)**: Run `git diff $MERGE_BASE..origin/main -- <conflicted-files>`. List every feature, function, import, prop, and hook main adds. Be exhaustive.

3. **Resolve each conflicted file**: For each file, read it with conflict markers and resolve:
   - Identify conflict type: **Additive** (both added different things → include both), **Structural** (one side refactored → keep refactored structure, port features), **Contradictory** (mutually exclusive → prefer main's structure, preserve PR's features)
   - Write the resolved file using Edit tool, then `git add <file>`
   - Resolution MUST keep all imports, functions, components, props, hooks, and event handlers from both branches

4. **Verify resolution**: Run `pnpm type-check` and `pnpm lint`. If either fails, fix errors and re-run. Launch a Haiku agent to verify no features were lost by comparing resolved files against both feature lists from step 2. Flag unused imports, uncalled functions, or arrays shorter than either branch had.

5. **Commit and push**: Commit with message listing resolved files and preserved features from each branch. Push to the PR branch.

6. **Report result**: Post a comment on the PR using `gh pr comment` with the RESULT_TEMPLATE below.

---

## Resolution Principles

| Principle | Description |
|-----------|-------------|
| Be additive | When in doubt, include code from both branches |
| Structure from main | If main refactored the code, use main's structure |
| Features from PR | Port all PR features into main's structure |
| No silent drops | Every import, function, and prop from both branches must exist in resolution |
| Fix forward | If resolution causes type errors, fix them rather than reverting |

## Common Structural Conflicts

| Conflict Type | Resolution |
|---------------|------------|
| Inline code → Hook extraction | Keep hook, add PR's features to hook |
| Component split | Keep split structure, ensure PR's logic exists in correct component |
| Renamed file/function | Use main's name, update PR's references |
| Added vs removed code | Keep the addition unless explicitly deleted by main |

## Failure Modes

If resolution cannot be completed:
1. Abort the merge: `git merge --abort`
2. Post comment explaining what failed
3. Exit with error code

## Templates

### COMMIT_MESSAGE_TEMPLATE

```
Merge main - resolve conflicts

Resolved conflicts in:
- {file_list}

Preserved features from PR branch:
- {pr_features}

Preserved features from main:
- {main_features}
```

### RESULT_TEMPLATE

```
## Conflicts Resolved

Automatically merged `main` and resolved conflicts.

### Files resolved:
| File | Resolution |
|------|------------|
| `{path}` | {resolution_type} |

### Verification:
- [x] Type check passed
- [x] Lint passed
- [x] All PR features preserved
- [x] All main features preserved

---
<sub>[Lleverage Dev Bot](https://github.com/lleverage-ai/dev-bot)</sub>
```
