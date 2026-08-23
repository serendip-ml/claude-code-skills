# grade-python-project

Evidence-based grading skill for Python projects. Produces a 0–10 score across 7 weighted
categories, backed by a verification command and its output for every deduction.

Loaded by [Claude Code](https://claude.ai/claude-code) via the surrounding
[claude-code-skills](../../) repository. See the [repository README](../../README.md) for
installation.

## What the skill does

- Discovers project layout, test infrastructure, and available quality tools.
- Runs baseline gates: LICENSE present, tests pass, core files exist, package imports.
- Grades each category on a 0–10 scale with severity-calibrated deductions.
- Prints a summary scorecard and the top issues to the conversation.
- Optionally saves the full report to `.GRADING.md` at the project root, with verified
  deductions, metrics, and an actionable roadmap.

## Usage

Inside a Python project directory in a Claude Code session:

    /grade-python-project

No arguments. The skill auto-detects package name, test command, and quality tooling.

## Categories and weights

| Category | Weight |
|---|---|
| Architecture & Design | 15% |
| Code Quality | 20% |
| Security | 15% |
| Testing | 15% |
| Documentation | 10% |
| Production Readiness | 15% |
| Dependencies & Compatibility | 10% |

Full per-category criteria, deduction rules, and severity guidance live in
[SKILL.md](SKILL.md).

## Evidence-based principles

Every deduction requires a verification command and its output. Pattern-matching
("looks incomplete") is not evidence. A category with no verified issues scores 10/10;
there is no "nothing is perfect" penalty.

Full principles are in
[../../rules/evidence-based-review.md](../../rules/evidence-based-review.md).

## Project type

Grading is scope-aware. Libraries and applications have different production-readiness
criteria (health checks, deployment guides, PyPI publishing, and so on). The skill reads
the target project's README to determine type before applying the rubric.

## Example output

See [examples/grade-output-pydantic.md](../../examples/grade-output-pydantic.md) for a
real-world grading of the pydantic library.

## License

MIT. See [LICENSE](../../LICENSE) at the repository root.
