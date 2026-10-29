# Installing pytest and choosing which tests run

**Covers:** pip, -v, --verbose, -x, -k, -m, --pdb

## From installation to a focused run {#overview}
Once you have written tests, you will spend a good part of your time running them in different ways: all of them, only one, only the failing ones, or only the fast ones. A small set of command-line flags covers almost every case. We start with the installation, and then go through the flags that narrow down a run or explain it.

## pip {#pip}
pip is Python's package installer. It is the command-line tool that downloads libraries from the Python Package Index and adds them to your environment. To get pytest, run `pip install pytest`. It is good practice to do this inside a virtual environment, which keeps the packages of one project apart from the others, so their versions do not clash. You can check the result with `pytest --version`.

You install plugins the same way, for example `pip install pytest-cov`. List what you need in a requirements file, so that other people get the same tools.

## -v {#v}
By default pytest prints a single dot for each test. `-v` changes that to one line per test with its result, such as `test_login.py::test_login_ok PASSED`. Use it when you want to know exactly which tests ran. If you repeat it, `-vv`, you see even more, such as full values in failures.

## --verbose {#verbose}
`--verbose` is the long form of `-v` and does exactly the same. Many pytest options have a short form for typing and a long form for readability. In scripts and configuration files, the long form is easier to understand later.

## -x {#x}
`-x` stops the whole run after the first failing test. It helps when one basic problem makes dozens of tests fail for the same reason: you fix the first failure, and the rest may pass. A variant, `--maxfail=3`, stops after three failures.

## -k {#k}
`-k` runs only the tests whose names match an expression. `pytest -k login` runs every test with "login" in its name, and the expression can combine words: `-k "login and not slow"`. It matches test names, class names and file names, and it ignores case.

It is the quickest way to run one test while you work on it. For a single exact test, you can also give its full path: `pytest test_login.py::test_login_ok`.

## -m {#m}
`-m` runs only the tests that carry a given marker. If some tests are marked `@pytest.mark.slow`, then `pytest -m slow` runs those and nothing else, and `pytest -m "not slow"` runs everything else. Markers are labels that you attach to tests, and later articles explain them.

## --pdb {#pdb}
`--pdb` makes pytest start the Python debugger at the moment a test fails, so you can look at the variables and step through the code. At the debugger prompt, `p variable` prints a value, `l` lists the code around you, `n` goes to the next line and `q` quits. It is the fastest way to see what the program state really was.

## A worked example {#example}
Suppose a project has these tests, and you want to work on login without the slow ones:

```python
tests = ["test_login_ok", "test_login_bad", "test_logout", "test_signup_slow"]
selected = [t for t in tests if "login" in t and "bad" not in t]
print(selected)
# ['test_login_ok']
```

This code imitates the expression `-k "login and not bad"`. A name must contain "login" and must not contain "bad", so only `test_login_ok` remains.

Here is the command-line version, in a session:

```bash
pip install pytest
pytest -v
pytest -x -k login
pytest -m "not slow" --pdb
```

Let's see what each line does.

1. The first line installs pytest.
2. `pytest -v` runs everything and lists each test. You would see 4 lines and a summary such as "3 passed, 1 failed".
3. `pytest -x -k login` runs only the two login tests and stops at the first failure.
4. The last command skips the slow tests and drops into the debugger if anything fails.

Now think about time. Say the full suite takes 4 minutes and the login tests take 3 seconds. Running only the login tests 80 times a day while you work saves about 80 × 237 seconds, which is 18,960 seconds, or 5.3 hours. That is why narrowing down is worth learning early.

## A common mistake {#mistake}
A common slip is to use `-x` on a CI server and then see only the first failure when you needed the full list. Use it locally, and not in the final check. Another slip is to forget about `-s` and the debugger and add print statements instead. The debugger usually shows you more.

## Key points {#key-points}
- Install with `pip install pytest`, preferably in a virtual environment.
- `-v` lists each test. `-x` stops at the first failure.
- `-k` selects by name and `-m` selects by marker.
- `--pdb` opens the debugger at the failure.
