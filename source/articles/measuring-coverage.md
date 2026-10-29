# Measuring coverage with pytest-cov

**Covers:** coverage, pytest-cov, branch coverage, --cov, --cov-branch, --mock

## A number to read with care {#overview}
After you write tests, one question comes up: how much of my code do they actually exercise? A tool can answer it as a percentage. The number is useful, but it is also easy to over-trust. This article shows how to measure it with pytest, and what the answer does and does not tell you.

## coverage {#coverage}
Test coverage is the percentage of your code's lines, or branches, that actually ran while your test suite executed. A tool watches the program while the tests run and records which lines were reached. A report then lists, for each file, how many lines ran and which did not.

What it proves is narrow: a covered line was executed. It does not prove that anything checked its result. A test that calls a function and asserts nothing gives full coverage of that function and no protection at all. Low coverage is therefore informative, because it shows code that nothing tests, while high coverage is not proof of quality.

## pytest-cov {#pytest-cov}
pytest-cov is a plugin, installed separately with `pip install pytest-cov`, that adds the `--cov` option to pytest. It connects pytest with the `coverage` tool, so that a normal test run also produces a coverage report. It handles combining results when tests run in parallel, which a manual set-up does not.

## --cov {#cov}
`--cov` turns on coverage measurement and says which package to measure. `pytest --cov=mypackage` runs the tests and prints a table at the end: one row per file with the number of statements, the number missed and the percentage. Add `--cov-report=term-missing` to also list the missed line numbers, and `--cov-report=html` for a browsable report that colours the lines.

Measure your own code, not the tests themselves and not libraries, which is why you name the package. A line like `--cov-fail-under=85` makes the run fail when the total drops below 85 percent, which is a way to stop coverage from slipping.

## --cov-branch {#cov-branch}
`--cov-branch` switches on branch coverage in addition to line coverage. It checks, for each decision in the code, such as an `if`, whether both outcomes were taken during the tests.

## branch coverage {#branch-coverage}
Branch coverage is a stricter form of coverage that checks both sides of every `if`, loop and similar decision, not only whether the line ran. An `if x > 0: return 1` line is "covered" as soon as it runs once, even if the condition was always true. With branch coverage, the case where it is false must also be run, or the report shows a partial branch.

It finds the untested "else" paths, which are where many bugs hide, such as the empty list or the error case.

## --mock {#mock}
`--mock` is not a pytest-cov flag, and pytest has no such option. Mocking comes from separate tools: the standard library's `unittest.mock` or the pytest-mock plugin. If you type `--mock`, pytest will complain about an unrecognised argument. A similar-looking option that does exist is `--cov`.

## A worked example {#example}
A function with one decision:

```text
def grade(score):
    if score >= 50:
        return "pass"
    return "fail"
```

One test, `assert grade(80) == "pass"`, runs the `if` line and the first return. The last line, `return "fail"`, is never reached.

```python
total_lines = 4
lines_run = 3
print(round(lines_run / total_lines * 100))
# 75
```

How to read it:

1. The function has 4 lines that count: the `def`, the `if`, and the two returns.
2. The single test runs the `def`, the `if` and the first return, which is 3 of 4 lines, or 75 percent.
3. `--cov-report=term-missing` would list line 4 as missed.

Adding `assert grade(20) == "fail"` runs the last line, and line coverage is 100 percent.

Now the branch view. Suppose the code has 6 branches in all (3 decisions, each with two outcomes), and the tests take 5 of them. Branch coverage is 5 / 6, about 83 percent, even if every line ran. The missing branch is the case that a plain line report would have hidden. The command to see it:

```bash
pytest --cov=mypackage --cov-branch --cov-report=term-missing
```

A target such as 85 percent is a guide, and not a goal in itself. Chasing 100 percent by writing assert-free tests raises the number and protects nothing.

## A common mistake {#mistake}
Treating a high percentage as proof that the code works. Coverage measures what ran, not what was checked. Another mistake is measuring the wrong package or including the tests in the percentage, which inflates it.

## Key points {#key-points}
- Coverage is the share of lines or branches that ran during the tests. It shows what is untested, and not what is well tested.
- pytest-cov adds `--cov=package`. Add `--cov-branch` to check both sides of each decision.
- `--mock` is not a pytest option. Mocking is a separate tool.
- Use coverage to find gaps, and do not chase the number.
