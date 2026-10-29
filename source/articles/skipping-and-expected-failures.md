# Skipping tests and expecting failures

**Covers:** skip, skipif, xfail, XPASS, OS

## Not every test can pass today {#overview}
Real projects have tests that cannot run on some machines, tests for features that are not finished, and tests that show a known bug. If you delete them, you lose the information. If you leave them red, they hide real failures in the noise. pytest has three ways to keep such tests in the suite without breaking the build, and each of them states its reason in the report. Let's begin with the idea that makes one of them necessary.

## OS {#os}
An OS, or operating system, is the software that manages a computer's hardware and runs other programs, such as Windows, macOS or Linux. It matters here because many tests only make sense on one of them. A test that uses Unix file permissions cannot run on Windows, and one that needs a Windows-only library cannot run elsewhere. Python tells you which one you are on in `sys.platform`, with values such as `"win32"`, `"linux"` and `"darwin"`.

## skip {#skip}
`@pytest.mark.skip` tells pytest not to run a test at all, unconditionally. Use it for a test that is temporarily broken or not ready yet. Always give a reason, as in `@pytest.mark.skip(reason="waiting for the new API")`, because the reason appears in the report and tells the next person why. You can also skip from inside a test with `pytest.skip("reason")`, when you discover at run time that the conditions are not right.

## skipif {#skipif}
`@pytest.mark.skipif(condition, reason="...")` skips a test only when a condition you supply is true. The classic use is skipping on one operating system or Python version: `@pytest.mark.skipif(sys.platform == "win32", reason="needs POSIX paths")`. The test runs everywhere else. pytest evaluates the condition when it collects the tests, so it has to be something known at that moment.

## xfail {#xfail}
`@pytest.mark.xfail` marks a test as expected to fail, usually because of a known bug. pytest still runs it, but it reports a failure as `xfailed`, which does not turn the run red. The test documents the bug, and it shows up in the report every time.

The options are `reason="..."` for the explanation, `strict=True` to turn an unexpected pass into a failure, `raises=SomeError` to accept only a certain kind of failure, and `run=False` to skip running it.

## XPASS {#xpass}
XPASS, "unexpectedly passed", is what pytest reports when a test marked `xfail` actually passes. It probably means that the bug has been fixed and the mark is out of date. By default pytest only reports an XPASS, and that is easy to miss. With `strict=True` it counts as a failure, which forces someone to remove the mark. Many projects turn on strict mode for all xfails in the configuration with `xfail_strict = true`.

## A worked example {#example}
Here are two marked tests:

```text
import sys

import pytest


@pytest.mark.skipif(sys.platform == "win32", reason="uses POSIX paths")
def test_paths():
    assert "/tmp".startswith("/")


@pytest.mark.xfail(reason="known rounding bug", strict=True)
def test_rounding():
    assert round(2.675, 2) == 2.68


print(test_rounding.pytestmark[0].name)
# xfail
```

This is how it reads in a report:

| Test | On Linux | On Windows |
|---|---|---|
| `test_paths` | passed | skipped (reason: uses POSIX paths) |
| `test_rounding` | xfailed (known rounding bug) | xfailed |

Let's explain the xfail. `round(2.675, 2)` gives 2.67, not 2.68, because 2.675 cannot be stored exactly in binary floating point. It is stored as slightly less than 2.675, so it rounds down. The assertion therefore fails, which matches the mark, so the result is "xfailed" and the build stays green.

Now suppose a future change makes it pass. Because of `strict=True`, the report would show a failure with "XPASS(strict)". That forces someone to look and remove the mark. Without `strict`, it would show a quiet `XPASS`.

Count the outcome for a suite of 100 tests where 2 are skipped, 1 is xfailed and 97 pass. The report reads "97 passed, 2 skipped, 1 xfailed", and the exit status says success. If you had deleted those three tests, the report would say only "97 passed", and nobody would know about the other three.

## A common mistake {#mistake}
A common slip is to skip a test with no reason, or never to revisit skips. A growing pile of skipped tests is a hidden problem, so review them regularly. Another slip is to use `xfail` for a flaky test, one that fails only sometimes. A flaky test needs fixing, because an xfail on it will sometimes pass and cause noise.

## Key points {#key-points}
- `skip` never runs a test. `skipif` skips it only when a condition is true, such as a platform check.
- `xfail` expects a failure, runs the test and reports `xfailed`.
- XPASS means an expected failure passed. Use `strict=True` so that it gets noticed.
- Always give a reason, and review skips and xfails regularly.
