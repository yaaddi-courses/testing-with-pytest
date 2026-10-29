# How pytest finds your tests

**Covers:** test discovery, _test.py, _check.py, _spec.py, __init__

## No registration, only names {#overview}
In many frameworks you must list each test somewhere, or inherit from a base class so that the runner knows it exists. pytest avoids that. It looks through your project and picks up anything that follows a few naming rules. That is convenient, but it also means that a test with the wrong name is silently ignored, which is a classic source of tests that never ran.

## test discovery {#test-discovery}
Test discovery is how pytest finds your tests automatically, by following naming conventions, without any list or registration. When you run `pytest` with no arguments, it walks the directories, selects the files that look like test files, imports them and collects the functions and classes that look like tests.

The default rules are:
- Files named `test_*.py` or `*_test.py`.
- Functions whose names start with `test`.
- Classes whose names start with `Test`, containing methods whose names start with `test`.
- Inside classes, no `__init__` method.

The collected list is what runs. `pytest --collect-only` prints it without running anything, which is the way to check what pytest will actually see.

## _test.py {#test-py}
`_test.py` is a file-name ending that pytest recognizes, for example `login_test.py`. The other accepted form is a name that starts with `test_`, like `test_login.py`. The two are equal, so pick one and use it throughout the project.

## _check.py {#check-py}
`_check.py` is an ending that pytest does not collect by default. A file called `login_check.py` is ignored, so the tests in it never run, and you get no warning. The same goes for names such as `login_tests.py` with an `s`, and `checks.py`.

## _spec.py {#spec-py}
`_spec.py` is also not collected by default. The style comes from other tools, such as the JavaScript testing world, where "spec" files are normal. In pytest, only `test_*.py` and `*_test.py` are found. If you want a different pattern, set `python_files` in the configuration, for example `python_files = ["*_spec.py"]`.

## __init__ {#init}
`__init__` is the Python method that sets up a new object. pytest cannot collect a test class that defines one, because pytest creates the class itself, without arguments, and it would not know how to call a custom constructor. It issues a warning and ignores the class. Use set-up functions for preparation instead.

## A worked example {#example}
These file names are matched against the default patterns:

```python
import fnmatch

names = ["test_login.py", "login_test.py", "login_check.py", "login_spec.py", "conftest.py"]
patterns = ["test_*.py", "*_test.py"]
collected = [n for n in names if any(fnmatch.fnmatch(n, p) for p in patterns)]
print(collected)
# ['test_login.py', 'login_test.py']
```

How to read it:

1. `test_login.py` starts with `test_`, so it matches the first pattern.
2. `login_test.py` ends with `_test.py`, so it matches the second.
3. `login_check.py` and `login_spec.py` match neither, so they are silently ignored.
4. The fifth file is a special helper file that pytest loads for shared set-up, and it is not a test file.

Now the inside of a file:

```text
# test_cart.py
def test_total():            # collected: starts with "test"
    assert 2 + 2 == 4

def check_discount():        # ignored: does not start with "test"
    assert 1 == 2

class TestCart:              # collected: starts with "Test", no __init__
    def test_empty(self):
        assert [] == []

class TestOrder:             # skipped, with a warning: it has __init__
    def __init__(self):
        self.items = []
    def test_add(self):
        assert True
```

Count the tests: 2 are collected, `test_total` and `TestCart.test_empty`. `check_discount` and `TestOrder.test_add` never run. The one with the false assertion, `check_discount`, shows how easy it is to leave a failing check unnoticed.

The remedy takes two commands: run `pytest --collect-only` and compare the count with the number that you expect, and add `-v` to see the names.

## A common mistake {#mistake}
Naming a file `checks.py` or a function `check_x`, running pytest, seeing "no tests ran" or a green result and assuming that all is well. Always confirm that the number of tests matches your expectation. Another mistake is giving a test class an `__init__`. Use a set-up function.

## Key points {#key-points}
- pytest finds tests by name: `test_*.py` or `*_test.py` files, `test*` functions and `Test*` classes.
- Files ending in `_check.py` or `_spec.py` are not collected by default.
- A test class with `__init__` is ignored. Use set-up functions for preparation.
- `pytest --collect-only` shows what will really run.
