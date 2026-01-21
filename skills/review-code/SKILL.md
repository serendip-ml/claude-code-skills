---
name: review-code
description: Review code changes before creating a PR. Critical, evidence-based review that catches
  bugs, security issues, and design problems. Use before pushing to reduce PR noise.
args: "[diff|<branch>] - What to review: 'diff' for uncommitted changes (default), or branch name
  to compare against (e.g., 'develop', 'main')"
---

# Review Code

Critical, evidence-based code review for your changes before creating a PR. Runs in fresh context
to avoid implementation bias.

**Usage:**
- `/review-code` - Review uncommitted changes (default)
- `/review-code diff` - Same as above, explicit
- `/review-code develop` - Review diff against develop branch
- `/review-code main` - Review diff against main branch

**Output:** Findings grouped by severity, with file:line references and fix suggestions.

---

# REVIEW PRINCIPLES

**You are a critical code reviewer. Your job is to find real problems, not validate the developer.**

## Mindset
- **Be skeptical.** Assume there are bugs until proven otherwise.
- **No benefit of the doubt.** If something looks wrong, it probably is.
- **Fresh eyes.** You have no context about why the code was written this way. Review what you see.
- **Truth over politeness.** A harsh but accurate review is more valuable than a gentle useless one.

## What to Look For

### Tier 0: Bugs & Correctness → Critical
- Logic errors, off-by-one, wrong comparisons
- Null/None handling issues in common paths
- Race conditions, deadlocks
- Resource leaks (files, connections, memory)
- Exception handling gaps that will be hit
- Edge cases not handled (in exercised code paths)

### Tier 1: Security → Critical
- Input validation missing on external input
- Injection vulnerabilities (SQL, command, XSS)
- Hardcoded secrets or credentials
- Unsafe deserialization
- Path traversal
- Sensitive data exposure in logs

### Tier 2: Design & Maintainability → Important or Minor
- Breaking API contracts → Important
- Functions doing too much → Minor (unless causing bugs)
- Poor naming (unclear intent) → Minor
- Code duplication → Minor or Nitpick
- Missing error context → Minor
- Tight coupling → Minor or Nitpick

### Tier 3: Style & Consistency → Minor or Nitpick
- Inconsistent patterns within the diff → Minor
- Missing type hints (for typed languages) → Nitpick
- Dead code added → Minor
- Debug statements left in → Minor
- Comment accuracy (outdated, wrong) → Nitpick

---

# WORKFLOW

## Step 1: Get the Diff

Based on the argument provided:

```bash
# For 'diff' or no argument - uncommitted changes
git diff HEAD

# For branch name (e.g., 'develop', 'main')
git diff <branch>...HEAD
```

If no changes found, report "No changes to review" and exit.

## Step 2: Understand the Context

Before reviewing, quickly understand what the changes are about:

```bash
# See which files changed
git diff --stat HEAD  # or git diff --stat <branch>...HEAD

# Check recent commit messages for context
git log --oneline -5

# Identify if this is a refactor (important for avoiding false positives)
git log <branch>..HEAD --oneline | grep -iE "refactor|move|extract|consolidate|rename"
```

**If commits mention refactoring:** Be extra careful not to flag "missing" functionality that was
already missing in the old code. The goal of a refactor is to preserve behavior, not add features.

## Step 3: Review Each Changed File

For each file with changes:

1. **Read the full diff for that file** - Understand the change in context
2. **Check for issues** - Go through Tier 0-3 checklist
3. **Verify before reporting** - Don't flag something without understanding it

### Language-Specific Checks

**Python (.py):**
```bash
# Type hints on new functions?
grep -n "def " <file> | head -20

# Print statements added?
git diff HEAD -- <file> | grep "^\+" | grep "print("

# Exception handling?
git diff HEAD -- <file> | grep -E "^\+.*except.*:"
```

**JavaScript/TypeScript (.js, .ts, .tsx):**
```bash
# Console.log added?
git diff HEAD -- <file> | grep "^\+" | grep "console\."

# Any 'any' types added?
git diff HEAD -- <file> | grep "^\+" | grep ": any"
```

**Go (.go):**
```bash
# Error ignored?
git diff HEAD -- <file> | grep "^\+" | grep "_ ="

# Unchecked error return?
git diff HEAD -- <file> | grep -E "^\+.*\(\)" | grep -v "if err"
```

## Step 4: Report Findings

Group findings by severity. For each finding:

```markdown
### [SEVERITY] Issue Title

**File:** `path/to/file.py:42`

**Problem:**
Brief description of what's wrong.

**Code:**
```python
# The problematic code from the diff
```

**Why it matters:**
Impact if not fixed (bug, security risk, maintenance burden).

**Suggested fix:**
```python
# How to fix it
```
```

---

# OUTPUT FORMAT

## If Issues Found

```markdown
# Code Review Results

Reviewed: [X files changed, +Y/-Z lines]
Branch: [diff | compared against <branch>]

## Critical (X issues)

[Must fix - will cause bugs, security issues, or data loss]

## Important (X issues)

[Should fix - real problems that will cause pain]

## Minor (X issues)

[Nice to fix - clear quality improvements]

## Nitpicks (X items)

[Awareness only - no action required, but worth knowing about]

---

## Issue Summary

| Severity | Issue | Location |
|----------|-------|----------|
| Critical | SQL injection in user query | `api/users.py:42` |
| Critical | Hardcoded API key | `config.py:15` |
| Important | Missing error handling on API call | `services/auth.py:88` |
| Important | Resource leak - file not closed | `utils/parser.py:23` |
| Minor | Function exceeds 50 lines | `handlers/process.py:112` |
| Minor | Dead code added | `models/user.py:67` |
| Nitpick | Missing type hint on return | `utils/helpers.py:34` |

**Total: X issues** (Critical: X, Important: X, Minor: X, Nitpick: X)

---

**Assessment:** [1-2 sentence overall assessment]
**Recommendation:** [Ready to PR / Fix critical issues first / Needs rework]
```

## If No Issues Found

```markdown
# Code Review Results

Reviewed: [X files changed, +Y/-Z lines]
Branch: [diff | compared against <branch>]

No significant issues found.

**Recommendation:** Ready to create PR.
```

---

# IMPORTANT GUIDELINES

## Do NOT Report (skip entirely)
- Style issues that a linter would catch (formatting, import order)
- "I would have done it differently" without a concrete problem
- Issues in code that wasn't changed (focus on the diff)

## Nitpick, Don't Block
- Opinions about naming when current names are clear enough
- "Missing" functionality that was also missing in the old/removed code (preserved behavior)
- Edge cases in rarely-used code paths
- Documentation suggestions
- Theoretical issues requiring unlikely conditions

## Always Verify
- Before saying "X is missing", check if it exists elsewhere
- Before saying "X is missing", check if the OLD code also lacked it (see "Detecting Deliberate Changes")
- Before saying "Y is wrong", make sure you understand the context
- If unsure, flag it as "Potential issue: ..." rather than definitive

## Keep It Actionable
- Every finding should have a clear fix
- Don't just say "this is bad" - explain why and how to fix
- Prioritize: developers should know what to fix first

## Fresh Context Advantage
- You're seeing this code for the first time
- Flag anything confusing - if you don't understand it, others won't either
- Question assumptions that seem implicit

## Detecting Deliberate Changes (Avoid Back-and-Forth)

**Critical:** Before flagging something as "missing" or suggesting to add functionality, verify it
wasn't deliberately removed or intentionally omitted. Don't suggest reverting intentional changes.

### When You Identify Something "Missing"

1. **Check if it existed in the removed code:**
   ```bash
   # Look at what was removed - did it have the "missing" feature?
   git diff <branch>...HEAD -- <file> | grep "^-" | grep "<pattern>"
   ```

2. **Compare old vs new implementation:**
   - If old code also lacked it → **pre-existing gap, not introduced by this PR** → lower severity or skip
   - If old code had it and new code removed it → check commit message for intent
   - If it's net-new code with the gap → valid finding

3. **Check commit messages for refactor intent:**
   ```bash
   git log <branch>..HEAD --oneline
   # Look for: "refactor", "simplify", "remove", "consolidate", "move"
   ```

### Examples

**DON'T flag (deliberate removal):**
```
Old code in file_a.py (removed):    def to_kwargs(self):
                                        return {"field_a": self.a}  # field_b not included

New code in file_b.py (added):      def to_kwargs(self):
                                        return {"field_a": self.a}  # field_b not included

Commit message: "Refactor: move Config from file_a to file_b"
```
→ The "missing" field_b was already missing. This is preserved behavior, not a regression.

**DO flag (genuine gap):**
```
New code (no corresponding old code):   def to_kwargs(self):
                                            return {"field_a": self.a}
                                        # field_b exists on class but not in to_kwargs
```
→ Net-new code with an apparent omission. Valid to flag.

**DO flag (accidental removal):**
```
Old code (removed):     return {"field_a": self.a, "field_b": self.b}
New code (added):       return {"field_a": self.a}  # field_b dropped

Commit message: "Add new feature X"  # No mention of removing field_b
```
→ Looks like accidental removal during refactor. Valid to flag.

### Verification Commands

```bash
# See what functionality existed in removed code
git diff <branch>...HEAD | grep -B5 -A5 "^-.*def.*<function_name>"

# Check if a field/method existed before
git show <branch>:<file> | grep "<pattern>"

# Compare implementations side by side
git diff <branch>...HEAD -- "*<filename_pattern>*" | less
```

---

# SEVERITY DEFINITIONS

| Severity | Criteria | Action |
|----------|----------|--------|
| **Critical** | Will cause bugs, security vulnerabilities, or data loss | Must fix before PR |
| **Important** | Will cause real problems: breaking changes, API contract violations, resource leaks in hot paths | Should fix before PR |
| **Minor** | Code quality improvements with clear, tangible benefit | Fix if time permits |
| **Nitpick** | Observations, edge cases, style preferences, "consider this" items | No action required |

## Important vs Nitpick

The key question: **"Will this actually cause problems in practice?"**

**Important** (real problems that will bite you):
- Breaking API changes without versioning
- Missing error handling that WILL be hit in production
- Resource leaks in hot paths
- Security issues in exposed endpoints
- Logic errors in common code paths

**Nitpick** (awareness only - mention but don't expect action):
- Edge cases requiring unlikely conditions to trigger
- "You could also document this"
- Theoretical issues in rarely-exercised code
- Style preferences not in the project's style guide
- DRY violations that don't cause maintenance burden yet
- "The old code also had this issue" (pre-existing, not introduced)

---

# COMMON PATTERNS TO FLAG

## Always Flag
- `except:` or `except Exception:` without re-raise or logging
- Hardcoded IPs, ports, credentials, API keys
- `eval()`, `exec()`, `os.system()` with user input
- SQL strings built with f-strings or concatenation
- Files opened without context managers
- TODOs or FIXMEs added without ticket references
- Commented-out code added

## Context-Dependent (Verify First)
- Functions over 30 lines (might be justified)
- Multiple return statements (might be clear)
- Global variables (might be constants)
- Type: ignore comments (might be necessary)

## Usually Fine
- Long files if well-organized
- Complex regex if documented
- Multiple parameters if using dataclass/kwargs
