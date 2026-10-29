# What pytest is, and how it differs from unittest

**Covers:** pytest, assert, unittest, TestCase, assertEqual, AssertionError, True, False

## Tests that read like ordinary Python {#overview}
Python ships with a testing tool of its own, yet many people choose a different one. The difference is mostly about how much ceremony it takes to write a single test. In this article we look at the tool most Python projects now use, at the plain keyword at its heart, and at the older, class-based style it is usually compared with.

## unittest {#unittest}
unittest is Python's built-in testing framework, and the one pytest is most often compared with. Its style comes from the Java tool JUnit. Every test is a method in a class that inherits from a base class, and you make checks with special methods. It works, and it is always available without installing anything, but it takes more writing to get the same result.

## TestCase {#testcase}
TestCase is the base class for tests in the unittest style. You create a class that inherits from `unittest.TestCase`, add methods whose names start with `test`, and the framework finds and runs them. The class can also define `setUp` and `tearDown` methods that run before and after each test. pytest can run TestCase classes too, so you can move an old project over gradually.

## assertEqual {#assertequal}
assertEqual is a unittest method that checks that two values are equal, as in `self.assertEqual(add(2, 3), 5)`. It belongs to a family: `assertTrue`, `assertIn`, `assertRaises` and many more, and a learner has to remember which one fits. In pytest you use none of these, because a plain comparison does the same job.

## assert {#assert}
assert is a plain Python keyword, and it is not special to pytest. It checks a condition, and if the expression after it is false, it raises an error and the test fails. You can add a message after a comma: `assert total == 5, "total should be 5"`.

There is one thing to know. Python can run with the `-O` option, which removes all assert statements, so never use assert for checks that must happen in production code, such as validating user input. Tests are safe, because they are not run that way.

## AssertionError {#assertionerror}
AssertionError is the exception that Python raises when an assert statement fails. A test runner catches it and reports the test as failed. You rarely handle it yourself. It carries the message you wrote, if any, and pytest adds more detail by showing the values involved.

## True {#true}
True is Python's boolean value for yes. An assert on it passes: `assert True` does nothing. In practice you assert expressions that give True, such as `x > 0`. Python also treats many other values as true: any non-empty string, any non-zero number and any non-empty list.

## False {#false}
False is the boolean value for no, and an assert on it fails the test. Python treats these values as false: False itself, None, zero, the empty string and empty collections such as `[]` and `{}`. This matters because `assert result` fails when `result` is an empty list, even though the list is not literally `False`.

## pytest {#pytest}
pytest is a Python testing framework that lets you write tests as plain functions, using the built-in assert keyword, instead of classes and special methods. You name a function so that it starts with `test_`, put it in a file whose name starts with `test_`, and run the command `pytest`. It finds the tests by itself.

It shows detailed failure messages, supports set-up code through fixtures, can run one test with many inputs, and has a large collection of plugins. Holger Krekel began it in the 2000s, and it is now the most widely used Python testing tool.

## A worked example {#example}
Here is the same check in both styles.

unittest:

```text
import unittest

class TestAdd(unittest.TestCase):
    def test_add(self):
        self.assertEqual(add(2, 3), 5)

if __name__ == "__main__":
    unittest.main()
```

pytest:

```python
def add(a, b):
    return a + b


def test_add():
    assert add(2, 3) == 5


def test_add_fails():
    try:
        assert add(2, 3) == 6, "2 + 3 should be 5"
    except AssertionError as error:
        return str(error)


test_add()
print(test_add_fails())
print(bool([]), bool([0]), bool(""), bool("False"))
# 2 + 3 should be 5
# False True False True
```

Let's read it.

1. The pytest check takes 2 lines, against 6 for unittest if you count its import and its main guard. It needs no class, no import and no special method.
2. `test_add` passes silently. The second test makes an assert fail on purpose and catches the AssertionError, so we can print its message.
3. The last line shows truthiness. An empty list is false, but `[0]` is true because it has one item. An empty string is false, and the string `"False"` is true, because it is not empty.

Now count the difference over a project of 500 tests. If unittest needs about 3 extra lines per test for the class, the method and the assertion call, that is roughly 1,500 lines of structure that pytest does not need.

## A common mistake {#mistake}
A common slip is to assume that `assert` is a function and to write `assert(a == b, "message")`. The parentheses make a tuple, which is always true, so the test never fails. Write it without parentheses around both parts. Another slip is using assert in production code for validation, since it can be turned off.

## Key points {#key-points}
- pytest tests are plain functions that use the built-in assert keyword. unittest uses classes and methods such as assertEqual.
- A failed assert raises AssertionError, and pytest explains the values.
- False, None, zero and empty collections count as false, so check what you assert.
- Never rely on assert for checks in production code, because `-O` removes it.
