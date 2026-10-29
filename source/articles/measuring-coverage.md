# Measuring coverage with pytest-cov

**Covers:** coverage, pytest-cov, branch coverage, --cov, --cov-branch, --mock

## A number to read with care {#overview}
After you write tests, a natural question follows: how much of my code do they actually exercise? A tool can answer it with a percentage. The number is useful, and it is also easy to over-trust. In this article we measure it with pytest, and then look at what the answer does and does not tell you.

## coverage {#coverage}
Test coverage is the percentage of your code's lines, or branches, that ran while your test suite executed. A tool watches the program while the tests run and records which lines were reached. A report then lists, for each file, how many lines ran and which did not.

What it proves is narrow. A covered line was executed, and that is all. It does not prove that anything checked the result. A test that calls a function and asserts nothing gives full coverage of that function and no protection at all. So low coverage is informative, because it shows code that nothing tests, while high coverage is not proof of quality.

## pytest-cov {#pytest-cov}
pytest-cov is a plugin you install separately with `pip install pytest-cov`. It adds the `--cov` option to pytest and connects pytest with the `coverage` tool, so an ordinary test run also produces a coverage report. It also combines the results when tests run in parallel, which a manual set-up does not do.

## --cov {#cov}
`--cov` switches coverage measurement on and names the package to measure. Running `pytest --cov=mypackage` runs the tests and prints a table at the end, with one row per file: the number of statements, the number missed and the percentage. Add `--cov-report=term-missing` to list the missed line numbers as well, and `--cov-report=html` for a browsable report that colours the lines.

You measure your own code, not the tests and not the libraries, which is why you name the package. A flag like `--cov-fail-under=85` makes the run fail when the total drops below 85 percent, which is a way to stop coverage from slipping.

## --cov-branch {#cov-branch}
`--cov-branch` adds branch coverage on top of line coverage. For each decision in the code, such as an `if`, it checks whether the tests took both outcomes.

## branch coverage {#branch-coverage}
Branch coverage is a stricter kind of coverage. It checks both sides of every `if`, loop and similar decision, not only whether the line ran. The line `if x > 0: return 1` counts as "covered" the first time it runs, even if the condition was always true. With branch coverage, you must also run the case where it is false, or the report shows a partial branch.

This matters because it finds the untested "else" paths, and many bugs hide there, such as the empty list or the error case.

## --mock {#mock}
`--mock` is not a pytest-cov flag, and pytest has no such option. Mocking comes from separate tools: `unittest.mock` in the standard library, or the pytest-mock plugin. If you type `--mock`, pytest complains about an unrecognised argument. A similar-looking option that does exist is `--cov`.

## A worked example {#example}
Here is a function with one decision:

```text
def grade(score):
    if score >= 50:
        return "pass"
    return "fail"
```

Suppose you have one test, `assert grade(80) == "pass"`. It runs the `if` line and the first return. The last line, `return "fail"`, is never reached.

```python
total_lines = 4
lines_run = 3
print(round(lines_run / total_lines * 100))
# 75
```

Let's read it.

1. The function has 4 lines that count: the `def`, the `if`, and the two returns.
2. The single test runs the `def`, the `if` and the first return. That is 3 of 4 lines, or 75 percent.
3. `--cov-report=term-missing` would list line 4 as missed.

If you add `assert grade(20) == "fail"`, the last line runs and line coverage is 100 percent.

Now the branch view. Suppose the code has 6 branches in all (3 decisions, each with two outcomes), and the tests take 5 of them. Branch coverage is 5 / 6, about 83 percent, even if every line ran. The missing branch is the case a plain line report would have hidden. The command to see it:

```bash
pytest --cov=mypackage --cov-branch --cov-report=term-missing
```

A target such as 85 percent is a guide and not a goal in itself. If you chase 100 percent by writing tests without asserts, the number rises and nothing is protected.

## A common mistake {#mistake}
A common slip is to treat a high percentage as proof that the code works. Coverage measures what ran, not what was checked. Another slip is to measure the wrong package, or to include the tests in the percentage, which inflates it.

## Key points {#key-points}
- Coverage is the share of lines or branches that ran during the tests. It shows what is untested, and not what is well tested.
- pytest-cov adds `--cov=package`. Add `--cov-branch` to check both sides of each decision.
- `--mock` is not a pytest option. Mocking is a separate tool.
- Use coverage to find gaps, and do not chase the number.
