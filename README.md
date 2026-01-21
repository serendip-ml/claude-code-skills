# Claude Code Skills

[![License:
MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude
Code](https://img.shields.io/badge/Claude%20Code-Skills-blue)](https://claude.ai/claude-code)

A collection of evidence-based code analysis skills for [Claude
Code](https://claude.ai/claude-code).

## Skills

| Skill | Command | Purpose |
|-------|---------|---------|
| **Grade Python Project** | `/grade-python-project` | Comprehensive audit with 0-10 scores across 7 categories |
| **Review Code** | `/review-code` | Pre-PR review catching bugs, security issues, design problems |

### Grade Python Project

Rigorous, evidence-based grading of Python projects. Every deduction requires verification
(command + output).

**Categories (weighted):**
- Architecture & Design (15%)
- Code Quality (20%)
- Security (15%)
- Testing (15%)
- Documentation (10%)
- Production Readiness (15%)
- Dependencies & Compatibility (10%)

**Usage:**
```bash
/grade-python-project
```

**Output:** Scorecard + top issues, optionally saves full report to `.GRADING.md`

### Review Code

Critical code review for changes before creating a PR. Runs with fresh context to avoid
implementation bias.

**Severity tiers:**
- Critical: Bugs, security vulnerabilities, data loss
- Important: Breaking changes, resource leaks
- Minor: Code quality improvements
- Nitpick: Observations, style preferences

**Usage:**
```bash
/review-code              # Review uncommitted changes
/review-code develop      # Compare against develop branch
/review-code main         # Compare against main branch
```

**Output:** Findings grouped by severity + recommendation (Ready to PR / Fix issues first)

## Installation

Copy the entire repo to your Claude Code commands directory:

```bash
# Clone
git clone https://github.com/serendip-ml/claude-code-skills.git

# Copy to Claude Code
cp -r claude-code-skills ~/.claude/commands/
```

Or copy individual skills:

```bash
# Just grade-python
cp -r claude-code-skills/skills/grade-python-project ~/.claude/commands/

# Just review-code
cp -r claude-code-skills/skills/review-code ~/.claude/commands/
```

## Project Structure

```
claude-code-skills/
├── commands/                    # Slash command definitions
│   ├── grade-python-project    # /grade-python-project command
│   └── review-code             # /review-code command
├── skills/                      # Skill implementations
│   ├── grade-python-project/
│   │   └── SKILL.md            # Grade Python skill (894 lines)
│   └── review-code/
│       └── SKILL.md            # Review Code skill (368 lines)
├── rules/                       # Shared principles
│   └── evidence-based-review.md
├── examples/                    # Example outputs
│   └── grade-output-pydantic.md
├── LICENSE
└── README.md
```

## Philosophy

Both skills share core principles (see `rules/evidence-based-review.md`):

1. **Evidence-based only** - Every finding backed by command output
2. **Verification before judgment** - "Could be problematic" ≠ "is problematic"
3. **Severity calibration** - Match findings to actual impact
4. **Actionable output** - Every issue includes location and fix suggestion
5. **No "vibes-based" assessments** - If you can't prove it, don't claim it

## Requirements

- [Claude Code CLI](https://claude.ai/claude-code)
- Python project with standard structure (for `/grade-python`)
- Git repository (for `/review-code`)

## Examples

See `examples/` for real-world outputs:
- `grade-output-pydantic.md` - Grading the pydantic library (9.30/10)

## License

MIT License - see [LICENSE](LICENSE)
