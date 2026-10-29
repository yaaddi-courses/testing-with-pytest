# Testing that code raises exceptions and warnings

**Covers:** pytest.raises, excinfo, ValueError, BaseException, KeyError, pytest.warns, DeprecationWarning, UserWarning

## Good code fails well {#overview}
A function is not finished when it handles good input. It must also refuse bad input in a clear, predictable way. Tests should check that refusal as carefully as they check the successful result. pytest provides one tool for exceptions and a matching one for warnings.

## ValueError {#valueerror}
ValueError is the exception that Python raises when a value has the right type but is not acceptable. For example, `int("abc")` raises it, because a string is the right type for `int()`, but this string cannot be turned into a number. Your own code should raise it, with a clear message, when an argument is of the correct kind but out of range, such as a negative age.

## KeyError {#keyerror}
KeyError is the exception raised when you look up a dictionary key that does not exist, as in `{"a": 1}["b"]`. It is common in tests of code that reads settings or records, which should fail with a clear message when something is missing.

## BaseException {#baseexception}
BaseException is the root of Python's whole exception hierarchy. Everything else, including `ValueError` and `KeyError`, inherits from it, through `Exception`. It also covers exit signals such as `KeyboardInterrupt` and `SystemExit`. That is why catching it is rarely a good idea, and why `pytest.raises(BaseException)` is too broad for a meaningful test: almost anything would satisfy it.

## pytest.raises {#pytest-raises}
`pytest.raises` is a context manager, used as `with pytest.raises(ValueError):`, that asserts that the code inside the block raises that exception. If the block raises it, the test continues. If it raises nothing, or raises something else, the test fails.

Add `match=` to check the message with a regular expression: `pytest.raises(ValueError, match="positive")`. Be as specific as you can with the exception type. A test that accepts any exception would hide a bug that raises the wrong one.

## excinfo {#excinfo}
`excinfo` is the object that you get from `with pytest.raises(...) as excinfo:`. It holds the exception that was caught. Use `excinfo.value` for the exception object, `excinfo.type` for its class and `str(excinfo.value)` for its message. It is available after the `with` block ends, so you can make further assertions about the details.

## pytest.warns {#pytest-warns}
`pytest.warns` is the counterpart of `pytest.raises` for warnings. In `with pytest.warns(UserWarning):`, the test asserts that the code inside actually issues that warning. A warning does not stop the program, so a normal test would not notice it. This is how you check that a deprecated function really warns.

## DeprecationWarning {#deprecationwarning}
DeprecationWarning is the warning category for features that will be removed in a future version. Library authors use it to tell users, "stop using this, move to the new one". Python hides it by default outside of test runs, but pytest displays it, which is a good reason to read the warnings summary.

## UserWarning {#userwarning}
UserWarning is the default category for warnings that your own code issues with `warnings.warn("message")`. Use more specific categories where they exist, since the category lets users filter warnings.

## A worked example {#example}
Test a function that rejects a bad age and another that warns.

```text
import warnings

import pytest


def parse_age(text):
    age = int(text)
    if age < 0:
        raise ValueError("age must be positive")
    return age


with pytest.raises(ValueError, match="positive") as excinfo:
    parse_age("-5")
print(excinfo.type.__name__)
print(str(excinfo.value))


def old_api():
    warnings.warn("use new_api", DeprecationWarning)
    return 1


with pytest.warns(DeprecationWarning):
    old_api()
print("warned")
# ValueError
# age must be positive
# warned
```

How to read it:

1. `parse_age("-5")` converts the text to -5, which is less than zero, so the function raises `ValueError`.
2. `pytest.raises(ValueError, match="positive")` accepts it, because both the type and the message pattern match. If the function had raised `KeyError`, the test would fail.
3. Through `excinfo`, we read the exception's class name and message.
4. `pytest.warns(DeprecationWarning)` passes because `old_api()` issued exactly that warning. Without the warning, it would fail with "DID NOT WARN".

Two other inputs show why specificity matters: `parse_age("abc")` raises `ValueError` too, but with a different message ("invalid literal for int()"). Because the test uses `match="positive"`, it would fail for that case, which tells you that the wrong check is doing the rejecting.

## A common mistake {#mistake}
Using `pytest.raises(Exception)` or `BaseException`, which passes for almost any failure, including a typing mistake in the test itself. Name the exact type and, if possible, the message. Another mistake is putting more than one statement inside the `with` block, so that you cannot tell which one raised. Keep it to the single call under test.

## Key points {#key-points}
- `pytest.raises` asserts that code raises a given exception. Add `match=` for the message.
- `excinfo` holds the caught exception, with `.type`, `.value` and the message.
- `pytest.warns` does the same for warnings, such as `DeprecationWarning` or `UserWarning`.
- Be specific. Broad types such as `BaseException` make a test almost meaningless.
