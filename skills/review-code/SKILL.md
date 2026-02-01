---
name: review-code
description: Review code changes before creating a PR. Critical, evidence-based review that catches
  bugs, security issues, and design problems. Use before pushing to reduce PR noise.
args: "[ref] [default|full|paranoid] | help"
---

# Review Code

Critical, evidence-based code review for your changes before creating a PR. Runs in fresh context
to avoid implementation bias.

**Usage:**
```
/review-code                  # Uncommitted changes, 1 agent
/review-code main             # Diff against main, 1 agent
/review-code abc123f          # Diff against commit hash, 1 agent
/review-code full             # Uncommitted changes, 2 agents
/review-code main full        # Diff against main, 2 agents
/review-code paranoid         # Uncommitted changes, 3 agents
/review-code main paranoid    # Diff against main, 3 agents
/review-code abc123f paranoid # Diff against commit, 3 agents
/review-code help             # Show help
```

**Modes:**
| Mode | Agents | Use when |
|------|--------|----------|
| `default` | 1 | Most PRs - fast, thorough, cost-effective |
| `full` | 2 | Adds context analysis (git history, architecture) |
| `paranoid` | 3 | Maximum scrutiny (bugs + design + history agents) |

**Output:** Findings grouped by severity, with file:line references and fix suggestions.

---

# ARGUMENT PARSING

Parse arguments to determine ref and mode:

- **ref**: A git reference - branch name, tag, or commit hash (default: HEAD)
- **mode**: `default`, `full`, or `paranoid` (default: default)

**Parsing rules:**
1. If argument is `help` → Show HELP OUTPUT below and exit
2. If argument is `default`, `full`, or `paranoid` → That's the mode, ref is HEAD
3. If argument is anything else → It's a ref, check for second argument for mode
4. If no arguments → ref=HEAD, mode=default

**Examples:**
| Input | Ref | Mode |
|-------|-----|------|
| (none) | HEAD | default |
| `help` | - | show help |
| `full` | HEAD | full |
| `paranoid` | HEAD | paranoid |
| `main` | main | default |
| `main full` | main | full |
| `develop paranoid` | develop | paranoid |
| `abc123f` | abc123f | default |
| `v1.2.0 paranoid` | v1.2.0 | paranoid |

After parsing, proceed to the appropriate mode section below.

---

# HELP OUTPUT

If argument is `help`, display this and exit:

```
/review-code - Critical code review before PR

USAGE
  /review-code [ref] [mode]
  /review-code help

ARGUMENTS
  ref       Git reference to compare against (default: HEAD for uncommitted)
            Can be: branch (main), tag (v1.2.0), or commit hash (abc123f)

  mode      Review intensity:
              default   1 agent - Standard review, good for most PRs
              full      2 agents - Adds context analysis (history, architecture)
              paranoid  3 agents - Maximum scrutiny (bugs + design + history)

EXAMPLES
  /review-code                  Review uncommitted changes
  /review-code main             Review changes vs main branch
  /review-code abc123f          Review changes vs specific commit
  /review-code full             Uncommitted changes, 2 agents
  /review-code main paranoid    Changes vs main, 3 agents

OUTPUT
  Findings grouped by severity:
    Critical   - Must fix (bugs, security, data loss)
    Important  - Should fix (breaking changes, resource leaks)
    Minor      - Nice to fix (code quality)
    Nitpick    - Awareness only (edge cases, style)
```

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

# MODE ROUTING

Based on the parsed mode:

- **default** → Execute MODE: DEFAULT below (single-agent review)
- **full** → Execute MODE: FULL below (2 parallel agents, then merge)
- **paranoid** → Execute MODE: PARANOID below (3 parallel agents, then merge)

---

# MODE: DEFAULT

Single-agent review. Execute the following workflow directly.

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

## Step 2b: Check Git History for Red Flags

For each modified file, check if git history reveals context that affects your review:

```bash
# Recent activity on modified files (high churn = fragile code)
git log --oneline --since="3 months ago" -- <file> | head -10

# Look for bug fixes, workarounds, or warnings in commit messages
git log --oneline --all-match --grep="fix\|bug\|workaround\|hack\|revert" -- <file>

# Check who last touched the lines you're modifying and when
git blame -L <start>,<end> -- <file>
```

### What to Look For

1. **Recent bug fixes** - If a function was bug-fixed recently and you're modifying it, verify you're
   not undoing the fix:
   ```bash
   git show <commit-sha> -- <file>
   ```

2. **High churn** - File changed 5+ times in 3 months = fragile code. Extra scrutiny warranted.

3. **Workaround indicators** - Commit messages mentioning "workaround", "hack", or "temporary" for
   code you're touching. Check if the workaround is still needed.

4. **Reverts in history** - If this code was reverted before, understand why before changing it again.

### When History Reveals a Risk

- **Read the relevant commit** - `git show <sha>` to understand what was fixed and why
- **Verify no regression** - Confirm the current change doesn't reintroduce a fixed bug
- **Flag appropriately:**
  - **Important** if there's real risk of regression (modifying recently-fixed code without accounting for the fix)
  - **Nitpick** if it's just awareness ("FYI: this function was bug-fixed 2 weeks ago, ensure your change is compatible")

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

# MODE: FULL

Two parallel agents with different focuses, then merge findings.

## Step 1: Get the Diff

Same as MODE: DEFAULT Step 1. Get the diff and file list.

## Step 2: Launch Agents

Launch 2 agents in parallel using the Task tool (subagent_type="general-purpose"):

### Agent 1: Code Reviewer

Focus: Bugs, security, design, style (Tier 0-3)

```
You are a code reviewer. Review this diff for bugs, security issues, and design problems.

DIFF TO REVIEW:
[INSERT DIFF]

FOCUS AREAS:
- Tier 0 (Critical): Logic errors, null handling, race conditions, resource leaks
- Tier 1 (Critical): Injection, hardcoded secrets, path traversal
- Tier 2 (Important/Minor): Breaking API changes, poor structure, tight coupling
- Tier 3 (Minor/Nitpick): Inconsistent patterns, dead code, debug statements

INSTRUCTIONS:
1. Review each file in the diff
2. Check against the focus areas above
3. Verify issues before reporting (don't flag deliberate changes)

OUTPUT FORMAT:
For each issue:
  ### [SEVERITY] Title
  **File:** path/file.py:42
  **Problem:** What's wrong
  **Code:** The problematic code
  **Why it matters:** Impact
  **Fix:** How to fix

If no issues: "No issues found."
```

### Agent 2: Context Analyst

Focus: Git history, regression risks, architectural impact

```
You are a context analyst. Check git history for regression risks.

FILES MODIFIED:
[INSERT FILE LIST]

INSTRUCTIONS:
For each file, run:
  git log --oneline --since="3 months ago" -- <file> | head -10
  git log --oneline --grep="fix\|bug\|workaround\|hack\|revert" -- <file>

LOOK FOR:
- Recent bug fixes (risk of undoing them)
- High churn (5+ changes in 3 months = fragile)
- Workarounds that might still be needed
- Previous reverts

If concerning history found, read the commit: git show <sha>

OUTPUT FORMAT:
For each risk:
  ### [SEVERITY] Title
  **File:** path/file.py
  **History:** What git history shows
  **Risk:** What could go wrong
  **Verify:** What to check

Severities: Important (real regression risk), Nitpick (awareness only)

If no concerns: "No historical concerns found."
```

## Step 3: Merge Findings

After both agents complete:
1. Collect all findings
2. Deduplicate: same file + similar issue = keep one, note "flagged by both agents"
3. Output unified report using OUTPUT FORMAT below

---

# MODE: PARANOID

Three parallel agents with specialized focuses, then merge findings.

## Step 1: Get the Diff

Same as MODE: DEFAULT Step 1. Get the diff and file list.

## Step 2: Launch Agents

Launch 3 agents in parallel using the Task tool (subagent_type="general-purpose"):

### Agent 1: Bug Hunter

Focus: ONLY bugs and security (Tier 0-1). Ignore design and style.

```
You are a bug hunter. Find bugs and security issues ONLY. Ignore design and style.

DIFF TO REVIEW:
[INSERT DIFF]

MINDSET: "Assume this code is broken. Find the bug."

LOOK FOR (ONLY THESE):
- Logic errors, off-by-one, wrong comparisons
- Null/None handling issues
- Race conditions, deadlocks
- Resource leaks (files, connections not closed)
- Exception handling gaps
- Injection, hardcoded secrets, path traversal

OUTPUT FORMAT:
For each bug:
  ### [Critical] Title
  **File:** path/file.py:42
  **Bug:** What's wrong
  **Impact:** What breaks
  **Fix:** How to fix

If no bugs: "No bugs found."
```

### Agent 2: Design Reviewer

Focus: ONLY design and API issues (Tier 2-3). Ignore bugs.

```
You are a design reviewer. Find design and API issues ONLY. Ignore bugs.

DIFF TO REVIEW:
[INSERT DIFF]

MINDSET: "Assume the architecture is wrong. Find the flaw."

LOOK FOR (ONLY THESE):
- Breaking API contracts (changed signatures, removed fields)
- Functions doing too much (>30 lines, multiple responsibilities)
- Poor naming (unclear intent)
- Tight coupling
- Inconsistent patterns within the diff

OUTPUT FORMAT:
For each issue:
  ### [SEVERITY] Title
  **File:** path/file.py:42
  **Problem:** Design issue
  **Impact:** Maintenance burden
  **Fix:** How to improve

Severities: Important (breaking changes), Minor (maintainability)

If no issues: "No design issues found."
```

### Agent 3: History Analyst

Focus: ONLY git history and regression risks.

```
You are a history analyst. Find regression risks using git history.

FILES MODIFIED:
[INSERT FILE LIST]

MINDSET: "Assume this will break something that was working. Find the regression."

INSTRUCTIONS:
For each file:
  git log --oneline --since="3 months ago" -- <file> | head -10
  git log --oneline --grep="fix\|bug\|workaround\|hack\|revert" -- <file>

For concerning commits: git show <sha> -- <file>

LOOK FOR:
- Recent bug fixes on this code
- High churn (5+ changes = fragile)
- Workarounds/hacks in commit messages
- Previous reverts

OUTPUT FORMAT:
For each risk:
  ### [SEVERITY] Title
  **File:** path/file.py
  **History:** What git shows
  **Risk:** Regression potential
  **Verify:** What to check

Severities: Important (real risk), Nitpick (FYI)

If no concerns: "No historical concerns found."
```

## Step 3: Merge Findings

After all 3 agents complete:
1. Collect all findings
2. Deduplicate: same file + similar issue = keep one
3. If multiple agents flagged same issue, note it (higher confidence)
4. Output unified report using OUTPUT FORMAT below

---

# OUTPUT FORMAT

## If Issues Found

```markdown
# Code Review Results

Reviewed: [X files changed, +Y/-Z lines]
Mode: [default | full | paranoid]
Ref: [HEAD | compared against <ref>]

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
Mode: [default | full | paranoid]
Ref: [HEAD | compared against <ref>]

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
