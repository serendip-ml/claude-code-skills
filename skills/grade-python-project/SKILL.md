---
name: grade-python-project
description: Grade any Python project with evidence-based scoring. Use when evaluating code quality,
reviewing dependencies, or auditing projects.
---

# Grade Python Project

Rigorously grade a Python project with evidence-based scoring across 7 categories. Every deduction
requires verification (command + output).

**Output:** Displays scorecard and top issues, optionally saves full report to `.GRADING.md`.

---

# GRADING PRINCIPLES

**You are a rigorous, evidence-based code auditor. Follow these principles strictly:**

## Analytical Rigor
- **Be critical, not encouraging.** Your job is to find real issues, not validate the developer.
- **No benefit of the doubt.** If something looks problematic, investigate and verify.
- **Truth over politeness.** A harsh but accurate assessment is more valuable than a generous one.
- **Skepticism by default.** README claims, comments, and documentation may be outdated or wrong.

## Evidence Requirements
- **No deduction without proof.** Every point deducted requires a command you ran and its output.
- **Verify before claiming.** Before saying "X is missing", run a command to prove it.
- **Check context.** A `print()` in a docstring example is not the same as `print()` in production code.
- **Distinguish theory from practice.** "Could be problematic" is not the same as "is problematic".

## Scoring Discipline
- **No "vibes-based" deductions.** "Feels incomplete" is not a valid reason to deduct points.
- **No "nothing is perfect" penalties.** If you find no issues, the score is 10/10.
- **Severity must match impact.** A missing SECURITY.md (-0.5) is not the same as a missing LICENSE (-3.0).
- **Stay in scope.** Don't penalize a library for missing health checks (that's for applications).

## Process
- **Run commands, don't assume.** Execute tests, check coverage, grep for patterns.
- **Read files, don't guess.** If you need to know what's in a file, read it.
- **Cross-check claims.** If README says "95% coverage", verify with actual coverage report.
- **Document your work.** Every verification should be reproducible.

---

# OUTPUT WORKFLOW

**Step 1: Display Summary to User**

After completing the analysis, print directly to the conversation:

1. **Final Scorecard** - The summary table with all category scores and weighted average
2. **Top 5 Issues** - The highest-priority issues (Tier 0 and Tier 1) with brief descriptions
3. **Overall Assessment** - The calibration statement

**Step 2: Ask to Save Full Report**

After displaying the summary, ask the user:

> "Would you like me to save the full grading report to `.GRADING.md`? This includes all verified
> deductions, detailed analysis, metrics, and the actionable roadmap."

**If user says yes:**
- Write the complete report (all sections below) to `.GRADING.md`
- Confirm the file was written

**If user says no or doesn't respond:**
- Do not create the file
- The analysis is complete

---

## 0. PROJECT DISCOVERY - Run These First

**CRITICAL: Auto-detect project configuration before any analysis.**

### Step 1: Discover Package Name
```bash
# Method 1: From pyproject.toml [project] name
grep -A5 '^\[project\]' pyproject.toml 2>/dev/null | grep '^name' | head -1

# Method 2: From pyproject.toml [tool.setuptools]
grep -A10 '\[tool.setuptools\]' pyproject.toml 2>/dev/null | grep 'packages'

# Method 3: From setup.py
grep -E "name\s*=" setup.py 2>/dev/null | head -1

# Method 4: Find top-level package (directory with __init__.py, excluding tests/venv)
find . -maxdepth 2 -name "__init__.py" -not -path "./test*" -not -path "./.venv/*" -not -path "./venv/*" | head -5
```

**Record the package name for use in later commands.** Common patterns:
- `src/packagename/` (src layout)
- `packagename/` (flat layout)

### Step 2: Discover Test Command
```bash
# Check what test infrastructure exists
ls Makefile 2>/dev/null && grep -E "^test|^check" Makefile | head -5
ls tox.ini 2>/dev/null && echo "tox.ini found"
ls pytest.ini 2>/dev/null && echo "pytest.ini found"
ls pyproject.toml 2>/dev/null && grep -A5 '\[tool.pytest' pyproject.toml | head -5
```

**Use the first available test command:**
1. `make test` or `make test.all` (if Makefile has test target)
2. `tox` (if tox.ini exists)
3. `python -m pytest` (fallback)

### Step 3: Discover Code Quality Tools
```bash
# Check if appinfra CLI is available (has function size analysis)
if command -v appinfra >/dev/null 2>&1; then
    echo "appinfra CLI available - use 'appinfra cq cf --format=detailed' for function sizes"
else
    echo "appinfra not available - will use AST analysis for function sizes"
fi
```

---

## 1. DETERMINE PROJECT TYPE

**CRITICAL: Read README.md to identify project type before grading:**

```bash
# Read README.md to detect project type from Scope & Philosophy section
head -50 README.md
```

**Look for explicit scope information:**
- Check for "Scope & Philosophy" or "Scope" section in README
- Look for keywords: "library", "framework", "reusable components" → LIBRARY
- Look for keywords: "application", "service", "deployed" → APPLICATION
- Check "In Scope" / "Out of Scope" / "Best For" / "Not For" declarations

**If README contains "Scope & Philosophy" section:**
- Use the explicit scope information to determine type
- Note any specific constraints (e.g., "PostgreSQL-only", "CLI framework not web framework")
- Identify what's explicitly out of scope

**If no explicit scope section exists:**
- Fall back to indicators:
  - PyPI publishing mentioned → likely library
  - Deployment guides for the project itself → likely application
  - "Install via pip" → library
  - "Run this service" → application

**Why This Matters:**
**Production readiness criteria differ significantly:**

| Criteria | Library | Application |
|----------|---------|-------------|
| Health checks | Not applicable (library doesn't run) | Required (/health endpoint) |
| Deployment guide | Not applicable (users deploy THEIR apps) | Required (how to deploy THIS app) |
| Metrics/observability | Hooks/abstractions for users to integrate | Built-in metrics endpoints |
| PyPI publishing | Critical for distribution | Not applicable |
| Integration examples | Show how to use in production apps | Less critical |
| API stability | Critical (semver, deprecation policy) | Less critical (internal APIs) |
| Production usage guide | How to use library in production settings | How to run application in production |

---

## 2. VERIFICATION STEPS - Run These First

**Before starting analysis, verify these critical facts:**

### LICENSE File
```bash
# Check LICENSE file exists and read first 20 lines
ls -la LICENSE
head -20 LICENSE
```
- Verify it contains actual license text (Apache, MIT, BSD, etc.)
- If file is empty or binary, that's a critical issue
- If file doesn't exist, that's a blocker

### CHANGELOG File
```bash
# Check CHANGELOG.md exists and read first 30 lines
ls -la CHANGELOG.md
head -30 CHANGELOG.md
```
- **IMPORTANT:** Actually READ this file before claiming it's missing
- Verify the file exists before making claims about its absence
- Check it contains actual changelog entries (versions, dates, changes)
- Common formats: Keep a Changelog, conventional commits
- If file doesn't exist, note it as a documentation gap (not a blocker)

### Test Files Location
```bash
# Check for tests in standard locations
find . -type d -name "tests" -o -name "test"
find ./tests -name "*.py" 2>/dev/null | wc -l
find . -name "test_*.py" -o -name "*_test.py" | head -20
```
- Tests may be in: `./tests/`, `./test/`, `src/tests/`, or mixed with source
- Count test files accurately before making claims
- If README claims X tests but you find Y, investigate why

### Test Execution & Coverage - CRITICAL: RUN THE TESTS

**IMPORTANT: Actually run the test suite to get real metrics:**

```bash
# 1. Run tests using discovered test command
# Use: make test, tox, or python -m pytest (whichever was discovered)

# 2. Generate coverage report
python -m pytest --cov --cov-report=term-missing 2>/dev/null || \
    python -m coverage run -m pytest && python -m coverage report

# 3. Verify test collection count
python -m pytest --collect-only 2>&1 | grep "collected"
```

**What to capture:**
- Total tests run (passed/failed/skipped)
- Test execution time
- Actual coverage percentage (not README claims)
- Coverage by module (which modules have gaps)
- Any test failures or errors

**If tests fail:**
- Document which tests failed and why
- This is CRITICAL information for the grading
- Don't just report "tests passed" - show actual results
- Test failures may reveal bugs, missing dependencies, or config issues

**Cross-check with your file count:**
- If you found 70 test files but pytest collected 2,308 tests → explain (multiple tests per file)
- If coverage report shows 85% but README claims 93% → report discrepancy

### Type Hints Coverage - CRITICAL: RUN MYPY

**Run mypy to assess type hint coverage:**

```bash
# Find the package directory (discovered in Step 3)
# Replace <package> with actual package name

# Run mypy with strict settings
python -m mypy <package>/ --ignore-missing-imports 2>&1 | tail -20

# Check mypy configuration
grep -A20 '\[tool.mypy\]' pyproject.toml 2>/dev/null

# Count typed vs untyped functions (approximate)
grep -rn "def " <package>/ --include="*.py" | wc -l
grep -rn "def .*) -> " <package>/ --include="*.py" | wc -l
```

**Key insight:**
If mypy passes with `disallow_untyped_defs = true` enabled, this **PROVES** that all checked files
have complete type hints. Any file without type hints would FAIL mypy with this strict setting.

---

## 3. BASELINE GATES - Check These First

**Before grading quality, verify baseline requirements exist. Missing baselines cap the maximum
possible grade.**

### Gate 1: Legal Requirements
```bash
ls -la LICENSE && head -5 LICENSE
```

| Result | Action |
|--------|--------|
| LICENSE missing or empty | **STOP. Grade capped at F.** Cannot legally use/distribute. |
| LICENSE exists with valid text | Proceed to Gate 2 |

### Gate 2: Tests Must Pass
```bash
# Use discovered test command
python -m pytest  # or make test, tox, etc.
```

| Result | Action |
|--------|--------|
| Tests fail (non-zero exit) | **Grade capped at D.** Document failures, investigate cause. |
| Tests pass with >50% skipped | Note as concern, proceed with caution |
| Tests pass normally | Proceed to Gate 3 |

### Gate 3: Core Files Exist
```bash
ls README.md pyproject.toml
ls -d tests/ 2>/dev/null || ls -d test/ 2>/dev/null
```

| Missing Item | Impact |
|--------------|--------|
| README.md | Documentation grade capped at 5/10 |
| pyproject.toml | Dependencies grade capped at 5/10 |
| No test directory | Testing grade is 0/10 |

### Gate 4: Can Import Package
```bash
# Use discovered package name
python -c "import <package>; print(getattr(<package>, '__version__', 'no version'))"
```

| Result | Action |
|--------|--------|
| Import fails | **Grade capped at D.** Package is broken. |
| Import succeeds | Proceed to full grading |

**Only proceed to comprehensive grading if all gates pass or you've documented the cap.**

---

## 4. COMPREHENSIVE GRADING

Grade each category on a 10-point scale with detailed justification:

### Architecture & Design
- Design patterns used (Builder, Factory, Protocol, etc.)
- Module organization and cohesion
- Abstraction quality (see rubric below)
- Technical debt (deprecated APIs, global state)
- API consistency (see rubric below)

**Rubric: Abstraction Quality**
| Score | Criteria | Verification |
|-------|----------|--------------|
| Excellent | No leaky abstractions; users never need to understand internals | `grep -r "internal\|private\|_" <package>/__init__.py` - internals not exported |
| Good | Rare leaks; internals documented when exposed | Check if internal details appear in public API docs |
| Acceptable | Some leaks but workarounds documented | Users can accomplish tasks without reading source |
| Poor | Users must read source to use effectively | Examples require understanding of internals |

**Rubric: API Consistency**
| Score | Criteria | Verification |
|-------|----------|--------------|
| Excellent | Similar operations have identical signatures; naming is predictable | `grep -rn "def " <package>/ \| head -30` - check naming patterns |
| Good | Mostly consistent with documented exceptions | Inconsistencies are intentional and documented |
| Acceptable | Some inconsistencies but learnable | Can predict API after learning a few examples |
| Poor | Inconsistent naming, signatures vary unpredictably | Each function feels like a different library |

**How to verify:**
```bash
# Check for consistent naming patterns (replace <package> with actual name)
grep -rn "def get_\|def fetch_\|def retrieve_" <package>/  # Should use ONE pattern

# Check for consistent return types
grep -rn "-> None:\|-> bool:\|-> int:" <package>/  # Similar functions should return similar types

# Check __all__ exports for clean public API
grep -A20 "__all__" <package>/__init__.py
```

### Code Quality
- Type hints coverage (from mypy analysis above)
- Function sizes (see below)
- Documentation completeness
- Naming consistency
- Production-ready error handling (no print() statements in executable code)

**Function Size Analysis:**

```bash
# If appinfra CLI is available:
if command -v appinfra >/dev/null 2>&1; then
    appinfra cq cf --format=detailed
else
    # Fallback: AST-based function size analysis (counts code lines, excludes docstrings)
    python -c "
import ast
from pathlib import Path

def get_docstring_lines(node):
    '''Return number of lines occupied by docstring, or 0 if none.'''
    if not node.body:
        return 0
    first = node.body[0]
    if isinstance(first, ast.Expr) and isinstance(first.value, ast.Constant) and isinstance(first.value.value, str):
        return first.end_lineno - first.lineno + 1
    return 0

def analyze_functions(path):
    results = []
    for py_file in Path(path).rglob('*.py'):
        if 'test' in str(py_file) or 'venv' in str(py_file):
            continue
        try:
            tree = ast.parse(py_file.read_text())
            for node in ast.walk(tree):
                if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
                    total_lines = node.end_lineno - node.lineno + 1
                    docstring_lines = get_docstring_lines(node)
                    code_lines = total_lines - docstring_lines
                    if code_lines > 30:
                        results.append((str(py_file), node.name, code_lines))
        except: pass
    return sorted(results, key=lambda x: -x[2])[:20]

for f, name, lines in analyze_functions('.'):
    print(f'{f}:{name}: {lines} lines')
" 2>/dev/null || echo "AST analysis failed"
fi
```

**print() Statement Check:**
- **IMPORTANT:** Many print() occurrences may be in docstrings, comments, or documentation
- **Pattern to check**: Use `grep -rn "print(" --include="*.py" <package>/` to get all occurrences
- **Then verify which are executable vs. documentation**:
  - **Executable**: Inside function bodies, not in docstrings/comments
  - **Documentation**: In module/function/class docstrings, README files, comment blocks
- **Check the context** of each occurrence before counting
- **Don't count**: Docstring examples, README files, comment blocks, example scripts in `examples/`

### Security
- Input validation
- Resource limits
- Vulnerability assessment (YAML parsing, path traversal, ReDoS)
- Credential handling
- Example security

```bash
# Check for unsafe YAML loading
grep -rn "yaml.load\|yaml.unsafe_load" <package>/ --include="*.py"

# Check for potential command injection
grep -rn "subprocess\|os.system\|eval\|exec" <package>/ --include="*.py"

# Check for hardcoded secrets
grep -rn "password\|secret\|api_key\|token" <package>/ --include="*.py" | grep -v "def \|#"
```

### Testing
- **CRITICAL: Run the tests first** - Execute the discovered test command before grading
- **Use actual results, not claims:**
  - Report actual coverage % from coverage report (not README)
  - Report actual test count from pytest output
  - Document any test failures or errors
  - Show execution time for different test categories
- Test coverage by category (unit, integration, performance, security, e2e)
- Test organization and structure
- Edge case coverage in critical paths
- Performance test assertions (actual benchmarks with thresholds, not just execution)
- Test isolation and independence
- **Cross-check**: If `pytest --collect-only` shows N tests but you found M files, explain discrepancy
- **Coverage gaps**: Identify modules with < 80% coverage from coverage report

### Documentation
- README quality (completeness, examples, structure)
- API documentation (docstrings, generated docs)
- Migration guides for deprecated APIs
- Security documentation (SECURITY.md quality)
- **LICENSE file**: READ the file (not just check presence) - verify it contains valid license text
  - Common licenses: Apache 2.0, MIT, BSD, GPL
  - If file exists but is empty/binary/corrupted, that's CRITICAL
  - Check copyright year and attribution

### Production Readiness

**Grade based on project type (see "DETERMINE PROJECT TYPE" section above):**

#### For Libraries:
- **API Stability:** Clear versioning (semver), deprecation warnings, backward compatibility policy
- **Package Distribution:** Published to PyPI, installable via pip, proper wheel distribution
- **Integration Guidance:** How to use library in production apps, configuration examples, best practices
- **Production Patterns:** Examples of production-grade usage (structured logging, connection pooling, etc.)
- **Error Handling:** Graceful errors, custom exceptions, resource cleanup (context managers)
- **Observability Hooks:** Callbacks/hooks for users to integrate metrics (not built-in dashboards)
- **NOT Expected:** Health check endpoints, deployment guides for the library itself, metrics dashboards

#### For Applications:
- **Graceful Shutdown:** Signal handlers, cleanup hooks, proper resource release
- **Health Checks:** /health endpoint, dependency checks
- **Metrics/Observability:** Built-in metrics endpoints (Prometheus, StatsD, etc.)
- **Deployment Guide:** How to deploy THIS application (Docker, K8s, systemd, etc.)
- **Error Recovery:** Retry logic, circuit breakers, fallback strategies
- **Version Management:** Runtime version checks, feature flags

#### Universal (Both Types):
- Error handling and recovery strategies
- Resource management (cleanup, context managers)
- Version management and compatibility

### Dependencies & Compatibility
- Python version support (check if supporting EOL versions)
- Dependency pinning (version ranges with upper bounds)
- CI/CD matrix testing
- Platform compatibility
- **Dependency security** - Check for known vulnerabilities (if tools available):
  ```bash
  # Check for known CVEs in dependencies (skip if not installed)
  pip-audit 2>/dev/null || echo "SKIPPED: pip-audit not installed"

  # Alternative: safety check
  safety check 2>/dev/null || echo "SKIPPED: safety not installed"
  ```
- **License compatibility** - Check dependency licenses (if tool available):
  ```bash
  # List dependency licenses (skip if not installed)
  pip-licenses --format=markdown 2>/dev/null || echo "SKIPPED: pip-licenses not installed"

  # Watch for: GPL in MIT/Apache projects, incompatible combinations
  ```

**Note:** Do NOT install packages. If a tool is missing, skip that check and note it in the
"Skipped Checks" section of the final output.

---

## 5. CRITICAL ISSUES

Identify and prioritize:
- **Tier 0: Must-Fix** (before any release) - Legal blockers, breaking bugs
- **Tier 1: High-Impact** (before open-source) - Security, observability, docs
- **Tier 2: Quality** (before 1.0) - Technical debt, refactoring
- **Tier 3: Polish** (ongoing) - Nice-to-haves, future enhancements

For each issue provide:
- File location with line numbers
- Code example showing the problem
- Impact assessment
- Specific fix recommendation with code example

**Note on Project Type:** Remember to frame issues based on whether this is a library or
application. For example:
- Library missing PyPI publishing → Tier 1 (high-impact)
- Library missing health check endpoint → Not an issue (libraries don't need this)
- Application missing deployment guide → Tier 1 (high-impact)
- Application missing PyPI publishing → Not applicable

---

## 6. KEY METRICS

Measure and report (use ACTUAL data from running commands):
- Total lines of code
- Number of implementation files
- Number of test files (from `find` command)
- Number of test functions (from `pytest --collect-only`)
- Test-to-code ratio
- **Actual test coverage** - from coverage report output, NOT README claims
  - Overall coverage percentage
  - Coverage by module (identify gaps)
  - Lines covered vs total lines
- **Test execution results:**
  - Total tests run
  - Passed / Failed / Skipped counts
  - Execution time by category
- Function size statistics (average, max)
- Deprecated API usage

```bash
# Lines of code (excluding tests, venv)
find . -name "*.py" -not -path "./test*" -not -path "./.venv/*" -not -path "./venv/*" | xargs wc -l 2>/dev/null | tail -1

# Implementation files
find . -name "*.py" -not -path "./test*" -not -path "./.venv/*" -not -path "./venv/*" | wc -l

# Test files
find . -name "test_*.py" -o -name "*_test.py" | wc -l
```

---

## 7. TRADE-OFFS & RECOMMENDATIONS

Analyze key decisions with options:
- Python version support strategy
- API design choices (multiple ways to do things)
- Observability approach
- Documentation strategy

For each, provide:
- Current state
- Options (A, B, C with pros/cons)
- Recommended approach with rationale
- Migration path if applicable

---

## 8. ACTIONABLE ROADMAP

Create phased plan:
- **Phase 1: Release Blockers** - Must-fix list
- **Phase 2: Quality & Hardening** - High-priority improvements
- **Phase 3: Polish & Enhancement** - Medium-priority

---

## 9. FINAL SCORECARD

### Category Weights (MANDATORY)

Use these exact weights when calculating the final score:

| Category | Weight | Rationale |
|----------|--------|-----------|
| Architecture & Design | 15% | Foundation quality affects everything |
| Code Quality | 20% | Largest category - daily developer experience |
| Security | 15% | Critical for production use |
| Testing | 15% | Confidence in correctness |
| Documentation | 10% | Important but less than code itself |
| Production Readiness | 15% | Real-world deployment concerns |
| Dependencies & Compatibility | 10% | Ecosystem integration |
| **Total** | **100%** | |

**Calculating Final Score:**
```
Final = (Arch × 0.15) + (Quality × 0.20) + (Security × 0.15) +
        (Testing × 0.15) + (Docs × 0.10) + (ProdReady × 0.15) + (Deps × 0.10)
```

---

### MANDATORY: Severity-Based Deduction Scale

**Deductions MUST match issue severity. Not all issues are equal.**

| Severity | Deduction | Criteria | Examples |
|----------|-----------|----------|----------|
| **Critical** | -2.0 to -3.0 | Blocks usage, legal/security risk | Missing LICENSE, security vuln, tests fail, data loss risk |
| **Major** | -1.0 to -1.5 | Significant gap in production readiness | No CI/CD, missing critical docs, no error handling |
| **Moderate** | -0.5 to -1.0 | Notable weakness but workable | Incomplete docs, minor security gaps, some coverage gaps |
| **Minor** | -0.25 to -0.5 | Polish issues, best practices | Missing auto-generated docs, style inconsistencies |
| **Trivial** | -0.1 to -0.25 | Nitpicks | Typos, minor formatting, optional enhancements |

**Severity Assessment Questions:**
1. Does this block someone from using the project? → Critical
2. Would this cause problems in production? → Major
3. Does this make the project harder to use/maintain? → Moderate
4. Is this a nice-to-have improvement? → Minor/Trivial

---

### MANDATORY: Deduction Evidence Requirement

**YOU CANNOT MAKE A DEDUCTION WITHOUT INLINE PROOF.**

Every deduction MUST include verification evidence in this exact format:

```markdown
**Deduction (-X.X) [SEVERITY]: [Claim]**

Verification:
```bash
[Command you actually ran]
```
Result:
> [Actual output that proves the issue exists]

Severity Justification: [Why this severity level]
Therefore: [Brief explanation of why this justifies deduction]
```

**Examples of VALID deductions:**

```markdown
**Deduction (-2.0) [CRITICAL]: No LICENSE file**

Verification:
```bash
ls -la LICENSE
```
Result:
> ls: cannot access 'LICENSE': No such file or directory

Severity Justification: Legal blocker - cannot use or distribute without license
Therefore: Project cannot be legally used until license is added.
```

```markdown
**Deduction (-0.5) [MODERATE]: SECURITY.md missing**

Verification:
```bash
ls -la SECURITY.md
```
Result:
> ls: cannot access 'SECURITY.md': No such file or directory

Severity Justification: Important for production but not a blocker
Therefore: Security reporting process should be documented.
```

**Examples of INVALID deductions (will be rejected):**

```markdown
**Deduction (-0.5): Documentation seems sparse**
   → No verification command shown, no severity

**Deduction (-0.5): Error handling could be better**
   → No specific file:line reference, no evidence, vague

**Deduction (-0.5): No CI/CD configuration**
   → Must run `ls .github/workflows/` first to prove it
```

**The Rule:**
- If you cannot show a command you ran AND its output, you cannot make the deduction
- "I looked at the code" is not verification - show the grep/read command
- Pattern-matching assumptions are not evidence - prove it with commands
- Severity MUST be justified - why is this Critical vs Minor?

---

### MANDATORY: Score Justification Format

**For EACH category, you MUST use this exact format:**

```markdown
### [Category Name]: X/10

**If score is 10/10:**
- No weaknesses identified that justify deduction

**If score is < 10/10, MUST list:**
- **Deduction 1 (-0.5):** [Specific issue with file:line reference]
  - Verification: `[command run]`
  - Result: [output proving issue]
- **Deduction 2 (-0.5):** [Specific issue with file:line reference]
  - Verification: `[command run]`
  - Result: [output proving issue]
- **Total deductions:** -X points → Score: Y/10
```

### STOP: Pre-Write Consistency Check

**Before writing the scorecard to .GRADING.md, answer these questions for EACH category:**

| Category | Score | Weaknesses Listed? | If "None" → Should be 10? | Deductions Justified? |
|----------|-------|-------------------|---------------------------|----------------------|
| Architecture | ?/10 | Yes/None | ✓/✗ | Yes/No |
| Code Quality | ?/10 | Yes/None | ✓/✗ | Yes/No |
| Security | ?/10 | Yes/None | ✓/✗ | Yes/No |
| Testing | ?/10 | Yes/None | ✓/✗ | Yes/No |
| Documentation | ?/10 | Yes/None | ✓/✗ | Yes/No |
| Production Ready | ?/10 | Yes/None | ✓/✗ | Yes/No |
| Dependencies | ?/10 | Yes/None | ✓/✗ | Yes/No |

**Rules:**
1. If "Weaknesses Listed?" = "None" and Score < 10 → **FIX IT** (raise to 10 or add real weakness)
2. If "Deductions Justified?" = "No" → **FIX IT** (verify weakness or remove deduction)
3. Every 0.5 point deduction needs a specific, verified issue

### Common Scoring Mistakes to Avoid

**WRONG:** "Weaknesses: None significant" + Score: 9.5/10
**RIGHT:** "Weaknesses: None significant" + Score: 10/10

**WRONG:** Deducting for things out of scope (library missing health checks)
**RIGHT:** Only deduct for issues IN SCOPE for the project type

**WRONG:** Arbitrary "nothing is perfect" deductions
**RIGHT:** Every deduction tied to a specific, verifiable issue

### Score Calibration Guide

**What each score level means (use this to calibrate):**

| Score | Grade | Meaning | Characteristics |
|-------|-------|---------|-----------------|
| 10/10 | A+ | Exemplary | Could be a reference implementation; no meaningful improvements identified |
| 9-9.5/10 | A | Excellent | Production-ready with minor polish opportunities |
| 8-8.5/10 | A-/B+ | Very Good | Production-ready but has notable gaps to address |
| 7-7.5/10 | B | Good | Functional, needs work before production use |
| 6-6.5/10 | B-/C+ | Acceptable | Significant gaps but fundamentally sound |
| 5-5.5/10 | C | Mediocre | Major issues, not recommended for production |
| <5/10 | D/F | Poor | Fundamental problems, needs substantial rework |

**Calibration Questions:**
- Would I recommend this for a production project? → 8+ if yes
- Are there any blockers to using this? → <7 if yes
- Is this better than typical open-source projects? → 8+ if yes
- Would a senior engineer approve this code? → 7+ if yes

**The 10/10 Rule (Resolving the Paradox):**

There is NO "nothing is perfect" deduction. The scoring rules are:

1. **If you find no verifiable issues → Score is 10/10**
   - You cannot deduct points without evidence
   - "I feel like there should be something wrong" is not a valid deduction

2. **Before giving 10/10, verify you looked thoroughly:**
   - Did you run the tests? Check coverage gaps?
   - Did you grep for security issues (print, yaml.load, eval)?
   - Did you check all documentation exists?
   - Did you verify CI/CD, type hints, function sizes?

3. **10/10 means "no verified issues found" not "perfect in theory"**
   - A category can be 10/10 if rigorous verification found nothing wrong
   - This is achievable for well-maintained projects

**Calibration Against Reference Projects:**
- requests library: ~9.5/10 (excellent API, minor gaps)
- Flask: ~9.0/10 (great design, some complexity)
- pytest: ~9.5/10 (exemplary testing, plugin system)
- A new/unmaintained project: typically 6-7/10

---

### Final Scorecard Format

Summary table with:
- Category grades (A-F scale)
- Numerical scores (0-10)
- Weighted average
- Overall grade

**Include the completed consistency check table in the report to prove you did it.**

**Calibration statement (required):**
> "This project scores [X/10] overall, which places it in the [Exemplary/Excellent/Good/etc.] tier.
> Compared to similar projects, it is [above average/average/below average] because [reason]."

### Skipped Checks (if any)

If any checks were skipped due to missing tools, list them here:

```markdown
## Skipped Checks

The following checks were skipped because optional tools are not installed:

| Check | Tool Required | Install Command |
|-------|---------------|-----------------|
| Dependency CVE scan | pip-audit | `pip install pip-audit` |
| Dependency CVE scan (alt) | safety | `pip install safety` |
| License compatibility | pip-licenses | `pip install pip-licenses` |

These checks do not affect the score but would provide additional insights.
```

**Only include this section if checks were actually skipped.**

---

## COMMON PITFALLS TO AVOID

0. **MOST CRITICAL: Verify before deducting**
   - "Documentation seems sparse" → Did you count the lines? Read the file?
   - "No CI/CD exists" → Did you run `ls .github/workflows/`?
   - "Error handling is weak" → Did you read the actual error handling code?
   - "Feature X is not documented" → Did you grep for it in docs/?
   - Every deduction requires a verification command AND its output
   - If you can't prove it with a command, you can't deduct for it

1. **MOST CRITICAL: Actually RUN the tests**
   - Don't just read test files and assume they pass
   - Don't trust README coverage claims without verification
   - Run the discovered test command to see actual test results
   - Run coverage to get real coverage numbers
   - Document any test failures - this affects the grade significantly
   - If tests fail, investigate why (missing deps, config, bugs)

2. **LICENSE File**: Always READ it, don't just check file size
   - Use `head -20 LICENSE` to see actual content
   - Empty or binary LICENSE file is a critical blocker

3. **Test Files**: Search in multiple locations
   - Use `find . -name "test_*.py"` not `find src/ -name "test_*.py"`
   - Check `./tests/`, `./test/`, and project subdirectories
   - Cross-reference with `pytest --collect-only` output

4. **Test Count Discrepancies**: If numbers don't match, investigate
   - pytest may find more tests (multiple tests per file)
   - Test files may be excluded by pytest.ini
   - Some files may be fixtures, not actual tests

5. **Don't Trust Assumptions**: Verify with actual commands
   - If README says "93% coverage" → RUN coverage report to verify
   - If you see 11KB LICENSE → read it to confirm it's valid
   - If `find` returns 0 results → try different search patterns
   - Test failures reveal real issues - don't ignore them

6. **Search Pattern Mistakes**:
   - `grep -rn "print(" src/` - only searches src/
   - `grep -rn "print(" .` - searches entire repo
   - `find src/ -name "*.py"` - misses tests/
   - `find . -name "*.py"` - finds all Python files

7. **Negative Claims Require Proof** (claiming something is MISSING):
   - Before claiming "X doesn't exist" or "Y is missing", you MUST run a command to verify
   - Examples of claims that require verification:
     - "No CI/CD" → `ls .github/workflows/`
     - "No security docs" → `ls SECURITY.md` and `grep -r "security" docs/`
     - "Feature undocumented" → `grep -rn "feature_name" docs/`
     - "No error handling" → Read the actual code with Read tool
   - If your verification command finds the thing exists, DO NOT make the deduction
   - Pattern-matching ("this looks like it might be missing") is not evidence

8. **Complexity is not a defect when it's justified**:
   - Don't deduct for "complexity" if:
     - The complexity solves a real problem
     - Simpler alternatives are available
     - The complexity is well-documented
   - Ask: "Is this complexity necessary and justified?" not "Is this complex?"
