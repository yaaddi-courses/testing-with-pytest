# Installing pytest and choosing which tests run

**Covers:** pip, -v, --verbose, -x, -k, -m, --pdb

## From installation to a focused run {#overview}
Once you have written tests, you spend much of your time running them in different ways: all of them, just one, just the failing ones or just the fast ones. A small set of command-line flags covers almost every case. This article starts with the installation and then covers the flags that narrow down or explain a run.

## pip {#pip}
pip is Python's package installer, the command-line tool that downloads libraries from the Python Package Index and adds them to your environment. To get pytest, run `pip install pytest`. It is good practice to do this inside a virtual environment, which keeps the packages of one project apart from the others, so that versions do not clash. You can check the result with `pytest --version`.

Plugins are installed the same way, for example `pip install pytest-cov`. List what you need in a requirements file so that others get the same tools.

## -v {#v}
`-v` makes pytest print one line for each test with its result, instead of a single dot. You see `test_login.py::test_login_ok PASSED`. Use it when you want to know exactly which tests ran. Repeating it, `-vv`, shows even more, such as full values in failures.

## --verbose {#verbose}
`--verbose` is the long form of `-v`, and does exactly the same. Many pytest options have a short form for typing and a long form for readability. In scripts and configuration files, the long form is easier to understand later.

## -x {#x}
`-x` stops the whole run after the first failing test. It is useful when a basic problem makes dozens of tests fail for the same reason. You fix the first one, and the rest may pass. A variant, `--maxfail=3`, stops after three failures.

## -k {#k}
`-k` runs only the tests whose names match an expression. `pytest -k login` runs every test with "login" in its name, and the expression can combine words: `-k "login and not slow"`. It matches against test names, class names and file names, and ignores case.

It is the quickest way to run one test while you work on it. For a single exact test, you can also give its full path: `pytest test_login.py::test_login_ok`.

## -m {#m}
`-m` runs only the tests that carry a given marker. If some tests are marked `@pytest.mark.slow`, then `pytest -m slow` runs just those, and `pytest -m "not slow"` runs everything else. Markers are labels that you attach to tests, as the next articles explain.

## --pdb {#pdb}
`--pdb` makes pytest start the Python debugger at the moment a test fails, so that you can look at the variables and step through the code. At the debugger prompt, `p variable` prints a value, `l` lists the code around you, `n` goes to the next line and `q` quits. It is the fastest way to see what the program state really was.

## A worked example {#example}
A project has these tests, and you want to work on login without the slow ones:

```python
tests = ["test_login_ok", "test_login_bad", "test_logout", "test_signup_slow"]
selected = [t for t in tests if "login" in t and "bad" not in t]
print(selected)
# ['test_login_ok']
```

This code imitates the expression `-k "login and not bad"`: a name must contain "login" and must not contain "bad". Only `test_login_ok` remains.

The command line version, in a session:

```bash
pip install pytest
pytest -v
pytest -x -k login
pytest -m "not slow" --pdb
```

What each does:

1. The first line installs pytest.
2. `pytest -v` runs everything and lists each test, so you see for example 4 lines and a summary "3 passed, 1 failed".
3. `pytest -x -k login` runs only the two login tests and stops at the first failure.
4. The last command skips the slow tests and drops into the debugger if anything fails.

Think about time: if the full suite takes 4 minutes and the login tests take 3 seconds, then running only them, 80 times a day while you work, saves about 80 x 237 seconds, which is 5.3 hours. That is why narrowing down is worth learning early.

## A common mistake {#mistake}
Using `-x` on a CI server and so seeing only the first failure, when you needed the full list. Use it locally and not in the final check. Another mistake is forgetting `-s` or the debugger and adding print statements instead. The debugger usually shows more.

## Key points {#key-points}
- Install with `pip install pytest`, preferably in a virtual environment.
- `-v` lists each test. `-x` stops at the first failure.
- `-k` selects by name and `-m` selects by marker.
- `--pdb` opens the debugger at the failure.
