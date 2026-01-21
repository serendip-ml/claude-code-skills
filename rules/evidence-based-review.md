# Evidence-Based Review Principles

Shared principles for all code analysis skills in this repository. These rules ensure rigorous,
objective, and actionable assessments.

---

## Core Philosophy

**Be a rigorous auditor, not a cheerleader.**

Your job is to find real problems, not validate the developer. A harsh but accurate assessment is
more valuable than a generous useless one.

---

## Analytical Rigor

### Mindset
- **Be skeptical.** Assume issues exist until proven otherwise.
- **No benefit of the doubt.** If something looks problematic, investigate and verify.
- **Truth over politeness.** Accuracy matters more than making people feel good.
- **Fresh eyes.** Review what you see, not what you assume the developer intended.

### Verification First
- **No claim without proof.** Every finding requires evidence you can show.
- **Run commands, don't assume.** Execute tests, check coverage, grep for patterns.
- **Read files, don't guess.** If you need to know what's in a file, read it.
- **Cross-check claims.** If README says "95% coverage", verify with actual coverage report.

### Distinguish Theory from Practice
- **"Could be problematic" ≠ "is problematic."** Theoretical issues need real-world validation.
- **Check actual usage.** Before flagging a pattern, verify how it's actually used.
- **Context matters.** A `print()` in a docstring is different from `print()` in production code.

---

## Evidence Requirements

### For Negative Claims (Something is Missing)

Before claiming "X doesn't exist" or "Y is missing", you MUST run a command to verify:

| Claim | Required Verification |
|-------|----------------------|
| "No CI/CD" | `ls .github/workflows/` |
| "No security docs" | `ls SECURITY.md` and `grep -r "security" docs/` |
| "Feature undocumented" | `grep -rn "feature_name" docs/` |
| "No error handling" | Read the actual code |
| "No tests" | `find . -name "test_*.py"` |

If your verification finds the thing exists, DO NOT make the claim.

### For Positive Claims (Something is Wrong)

Every issue must include:
1. **Specific location** - file path and line number
2. **Evidence** - command run and its output
3. **Severity justification** - why this severity level
4. **Impact** - what will happen if not fixed

### Invalid Evidence
- "I looked at the code" - show the grep/read command
- "It seems like..." - prove it with commands
- "Feels incomplete" - not a valid finding
- Pattern-matching hunches without verification

---

## Severity Calibration

### Severity Levels

| Level | Criteria | Examples |
|-------|----------|----------|
| **Critical** | Blocks usage, legal/security risk | Missing LICENSE, security vuln, tests fail |
| **Major/Important** | Significant production impact | Breaking changes, resource leaks, no error handling |
| **Moderate/Minor** | Notable weakness, workable | Incomplete docs, coverage gaps |
| **Trivial/Nitpick** | Polish, nice-to-have | Typos, style preferences |

### Severity Assessment Questions

1. Does this block someone from using the project? → Critical
2. Would this cause problems in production? → Major
3. Does this make the project harder to use/maintain? → Moderate
4. Is this a nice-to-have improvement? → Minor/Trivial

### Common Mistakes

- **Overclaiming severity** - A missing optional doc is not Critical
- **Underclaiming severity** - A security vulnerability is not Minor
- **Scope creep** - Library missing health checks is not an issue (libraries don't run)

---

## Detecting Deliberate vs Accidental Changes

When reviewing changes, distinguish between:

### Deliberate Removal (Don't Flag)
```
Old code: return {"a": self.a}  # field_b not included
New code: return {"a": self.a}  # field_b not included
Commit: "Refactor: move Config to new file"
```
→ Pre-existing gap, preserved behavior. Not a regression.

### Accidental Removal (Do Flag)
```
Old code: return {"a": self.a, "b": self.b}
New code: return {"a": self.a}  # field_b dropped
Commit: "Add new feature"
```
→ Looks like accidental removal. Valid finding.

### Verification Commands
```bash
# What existed in removed code?
git diff <branch>...HEAD -- <file> | grep "^-" | grep "<pattern>"

# Check commit intent
git log <branch>..HEAD --oneline | grep -iE "refactor|move|remove"
```

---

## What NOT to Report

- **Style issues linters catch** - formatting, import order
- **Opinions without problems** - "I would have done it differently"
- **Unchanged code issues** - focus on what was modified
- **Scope-inappropriate issues** - libraries vs applications have different needs
- **Theoretical issues** - unlikely edge cases in rarely-used paths

---

## Actionability

Every finding must be actionable:

1. **Clear location** - exact file and line
2. **Concrete problem** - what is wrong
3. **Why it matters** - impact if not fixed
4. **How to fix** - specific suggestion

Don't just say "this is bad" - explain why and how to fix it.

---

## The 10/10 Rule

There is NO "nothing is perfect" penalty.

1. **No verified issues → Score is 10/10** (for grading skills)
2. **No issues found → "Ready to PR"** (for review skills)

Before declaring "no issues":
- Did you run the verification commands?
- Did you check for common problems (security, tests, docs)?
- Did you verify, not assume?

If rigorous verification found nothing wrong, that's a valid result.
