---
allowed-tools: Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh api:*)
description: Code review a pull request
disable-model-invocation: false
---

Provide a code review for the given pull request using inline comments.

Arguments:
- `--force` or `-f`: Skip eligibility checks and always provide a review

## Steps

1. **Eligibility check** (skip if `--force`): Use a Haiku agent to check if the PR is closed or already has your review. Draft PRs are allowed.

2. **Fetch PR details**: Get the PR diff, metadata, and head commit SHA using:
   ```bash
   gh pr view {number} --json title,body,files,headRefOid,baseRefName
   gh pr diff {number}
   ```
   Count files changed and identify affected services/packages.

3. **Find CLAUDE.md files**: Use a Haiku agent to locate all relevant CLAUDE.md files:
   - Root CLAUDE.md if it exists
   - Any CLAUDE.md files in directories containing modified files

4. **Review the changes**: Launch 3 parallel Opus agents:

   **Agent 1 (Logic & Bugs)**: Scan for logic errors, bugs, edge cases, and incorrect assumptions. Focus on:
   - Off-by-one errors
   - Null/undefined handling
   - Incorrect conditions
   - Missing error handling
   - Wrong return values

   **Agent 2 (Security)**: Check for security vulnerabilities with CONTEXTUAL understanding. Review the "Security Domain Knowledge" section below before flagging issues. Focus on:
   - Injection vulnerabilities (SQL, command, XSS)
   - Broken access control (missing ownership checks, IDOR)
   - Sensitive data exposure (logging secrets, hardcoded credentials)
   - Insecure deserialization
   - Authentication/authorisation flaws

   **Agent 3 (Compliance)**: Check for CLAUDE.md violations and anti-patterns:
   - Violations of explicit rules in CLAUDE.md
   - Anti-patterns specific to this codebase
   - Significant deviations from established patterns

   Each agent returns issues with: file path, line number, category, severity, description, and suggested fix if applicable.

   **HIGH SIGNAL ONLY** - Flag issues where:
   - Code will fail to compile or parse
   - Code will definitely produce wrong results
   - Clear security vulnerability with exploitable impact
   - Clear, unambiguous CLAUDE.md violations with exact rule quotes

   **Do NOT flag:**
   - Code style or quality concerns
   - Potential issues dependent on specific inputs or state
   - Subjective suggestions or improvements
   - Patterns that LOOK wrong but are standard practice (see Security Domain Knowledge)

5. **Validate issues**: For EACH issue from the review agents, launch a parallel Haiku agent to validate:
   - Re-read 50+ lines of surrounding context
   - Check the "Common False Positives" section - reject if it matches
   - Verify the issue is real AND introduced by this PR
   - Check if it's actually a bug vs intentional behaviour
   - Check if PR description explains this as intentional
   - Score confidence 0-100

   **Validation requirements:**
   - Score must be 80+ to pass
   - Validator must explain WHY this is a real issue in 1-2 sentences
   - If validator is uncertain, reject the issue

6. **Assign severity**: For each validated issue:
   - `critical`: Will cause data loss, security breach, or system failure
   - `high`: Will cause incorrect behaviour in common cases
   - `medium`: Will cause incorrect behaviour in edge cases
   - `low`: Minor issue, defence-in-depth improvement

7. **Post review**: Post the review using inline comments. You MUST:
   - Post a summary comment first
   - Post each issue as a separate inline comment on the specific line

   **Summary comment** (use `gh pr comment`):
   ```bash
   gh pr comment {number} --body "$(cat <<'EOF'
   ## Code Review

   {one_sentence_summary}

   | Severity | Category | File | Issue |
   |----------|----------|------|-------|
   | {severity} | {category} | `{file}:{line}` | {brief_description} |

   See inline comments for details.

   ---
   <sub>[Lleverage Dev Bot](https://github.com/lleverage-ai/dev-bot)</sub>
   EOF
   )"
   ```

   **Inline comments** (post EACH issue separately):
   ```bash
   # Get the head commit SHA first
   HEAD_SHA=$(gh pr view {number} --json headRefOid -q '.headRefOid')

   # For EACH issue, post a separate comment
   gh api repos/{owner}/{repo}/pulls/{number}/comments \
     -f body="$(cat <<'EOF'
   **{severity} - {category}**: {description}

   ```suggestion
   {corrected_code_if_applicable}
   ```

   <details>
   <summary>Why this matters</summary>

   {explanation_of_impact}

   </details>
   EOF
   )" \
     -f path="{file_path}" \
     -f line={line_number} \
     -f commit_id="$HEAD_SHA"
   ```

   **If NO issues found**, post only:
   ```bash
   gh pr comment {number} --body "$(cat <<'EOF'
   ## Code Review

   {one_sentence_summary}

   No issues found.

   ---
   <sub>[Lleverage Dev Bot](https://github.com/lleverage-ai/dev-bot)</sub>
   EOF
   )"
   ```

---

## Security Domain Knowledge

**CRITICAL**: Before flagging security issues, understand these common patterns.

### Tokens That Do NOT Need Hashing

These tokens are looked up by value (like a key), not compared after hashing:

| Token Type | Why No Hashing | Security Model |
|------------|----------------|----------------|
| Bearer tokens / API keys | Token IS the credential, looked up directly | Cryptographic randomness + expiry + HTTPS |
| OAuth state tokens | Looked up to validate OAuth flow | Short-lived, single-use |
| Session IDs | Looked up in session store | Cryptographic randomness + expiry |
| Magic login links | Looked up to authenticate user | Single-use + expiry + rate limiting |
| Password reset tokens | Looked up to validate reset request | Single-use + short expiry |
| Invitation tokens | Looked up to validate invitation | Single-use + expiry |

**Only hash credentials that are COMPARED, not looked up:**
- User passwords (compared via bcrypt/argon2)
- Permanent secrets that could be reused if leaked

### Standard Security Patterns (Not Vulnerabilities)

| Pattern | Why It's OK |
|---------|-------------|
| Environment variables accessed directly | Standard practice, secrets managed by infrastructure |
| No CSRF protection on API with bearer auth | Bearer tokens are not sent automatically like cookies |
| Rate limiting not in application code | Usually handled by infrastructure (nginx, API gateway) |
| No input validation on internal service calls | Trust boundary is at the edge, internal calls are trusted |
| Credentials passed in request body over HTTPS | HTTPS encrypts the body, this is standard OAuth practice |

---

## Common False Positives

**REJECT issues that match these patterns:**

### Security False Positives
- Bearer tokens stored without hashing → **REJECT** (see above)
- Session IDs not encrypted at rest → **REJECT** (standard practice)
- Single-use tokens without hashing → **REJECT** (see above)
- Missing rate limiting → **REJECT** (infra concern, not code review)
- Missing CSRF on bearer-auth endpoints → **REJECT** (not needed)
- Open redirect without sensitive action → **REJECT** (low impact)
- DoS via resource exhaustion → **REJECT** (infra concern)
- Generic "input validation missing" without specific exploit → **REJECT**

### Logic False Positives
- Intentional early returns that look like missing error handling → **REJECT**
- Optional parameters that appear unused (but are used conditionally) → **REJECT**
- Type assertions that look unsafe but are guarded elsewhere → **REJECT**
- Code that looks wrong but matches existing codebase patterns → **REJECT**
- Changes that are intentional based on PR description → **REJECT**

### Pre-existing Issues
- Issues in unchanged code → **REJECT**
- Issues in code that was only moved, not modified → **REJECT**
- Style issues that a linter would catch → **REJECT**

---

## Categories

Use these exact category names:

| Category | Severity Guide | Use For |
|----------|----------------|---------|
| `security` | Usually critical/high | Injection, broken access control, data exposure |
| `logic` | Varies by impact | Logic errors, incorrect conditions, wrong assumptions |
| `error-handling` | Usually medium | Missing or incorrect error handling |
| `race-condition` | Usually high | Concurrency issues, TOCTOU |
| `compliance` | Usually low/medium | CLAUDE.md violations, anti-patterns |

---

## Notes

- **Inline comments are mandatory** - Never dump all issues in the summary
- Post each inline comment separately to ensure they appear on the correct lines
- Only include `suggestion` block if you have a concrete, complete fix
- Focus on significant issues only, not style or nitpicks
- Do not run builds, tests, or type checks (CI handles this)
- When uncertain, do NOT flag the issue - false positives erode trust
