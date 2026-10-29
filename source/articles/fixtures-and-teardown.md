# Fixtures: reusable setup and teardown

**Covers:** fixture, decorator, teardown, close(), yield fixture

## Setting the stage for a test {#overview}
Think about the last test you wrote. It probably needed something prepared first: a user record, a temporary file, an open connection. If you write that preparation at the top of every test, the code repeats and hides what each test is really about. pytest solves this with fixtures, small functions that prepare things and hand them to any test that asks. In this article we see how fixtures are declared, how tests use them and how they clean up.

## decorator {#decorator}
You will see `@something` above a function definition very often in pytest. That is a decorator. The line above `def` is an ordinary Python expression that receives the function and returns a version of it, with extra behaviour or a record in a registry. For example, `@pytest.fixture` above a function tells pytest that the function is a fixture. Decorators appear throughout pytest: in fixtures, labels and parametrization.

## fixture {#fixture}
A fixture is a set-up function that you mark with the `@pytest.fixture` decorator, so that a test can use it by naming it as a parameter. When pytest sees the parameter name, it finds the fixture with that name, runs it and passes its return value to the test.

```text
import pytest

@pytest.fixture
def user():
    return {"name": "Ana", "role": "admin"}

def test_role(user):
    assert user["role"] == "admin"
```

Notice that the test does not call `user()`. It asks for it, which is called dependency injection. A fixture can ask for other fixtures in the same way, so you build set-up from small, named pieces. By default pytest runs a fixture again for each test, so every test gets a fresh object.

## close() {#close}
`close()` is the method that releases a file, a network connection or a database handle when you have finished with it. If you forget it, resources leak: files stay locked, connections run out and data may not be written. A fixture that opens such a resource should also arrange for `close()` to be called afterwards, even when the test fails.

## teardown {#teardown}
Teardown is the cleanup a fixture does after a test has finished with it: closing a file, deleting a temporary record, stopping a server. It puts the world back the way it was, so one test's leftovers cannot disturb the next. The word comes from the set-up and teardown pairs of the older xUnit frameworks.

## yield fixture {#yield-fixture}
A yield fixture uses `yield` where an ordinary fixture would use `return`. The code before the `yield` is the set-up, the yielded value goes to the test, and the code after the `yield` is the teardown.

```text
@pytest.fixture
def connection():
    conn = open_connection()
    yield conn
    conn.close()
```

pytest runs the part after the yield when the test finishes, whether it passed or failed. That is the main benefit. The cleanup sits next to the set-up, in the same function, and a failing assertion in the test cannot bypass it.

There is one limit. If the set-up fails before the `yield`, the cleanup does not run. So if a fixture creates several things, give each one its own cleanup, or split them into fixtures that depend on one another.

## A worked example {#example}
This program imitates what pytest does with a yield fixture, so that you can see the order of events.

```python
def resource():
    print("setup")
    yield "connection"
    print("teardown")


gen = resource()
value = next(gen)
print(value)
try:
    next(gen)
except StopIteration:
    pass
# setup
# connection
# teardown
```

Let's follow it.

1. `next(gen)` runs the function up to the `yield`. That prints "setup" and produces the value `"connection"`.
2. The test would run at this point, holding that value. Here we print it instead.
3. The second `next(gen)` resumes the function after the `yield`. It prints "teardown", the function ends and `StopIteration` is raised, which we ignore.

That is the pattern pytest follows around a test: ask for the value, run the test, and resume the fixture so that it can clean up.

Now compare two styles for a test that needs a file:

| | In every test | As a yield fixture |
|---|---|---|
| Lines of set-up and cleanup per test | About 4 | 0 |
| Cleanup if the assertion fails | Not run unless wrapped in `try` and `finally` | Always runs |
| For 30 tests that need the file | 120 repeated lines | One 5-line fixture |

With the fixture, each of the thirty tests holds only the assertion that matters, and you fix the cleanup in one place.

## A common mistake {#mistake}
A common slip is to call a fixture function directly, as `user()`, instead of asking for it as a parameter. pytest forbids it. Another slip is to write the cleanup after an assertion in the test itself. If the assertion fails, the cleanup never runs. Put it in the fixture, after the yield.

## Key points {#key-points}
- A decorator such as `@pytest.fixture` registers or wraps the function below it.
- A fixture is requested by naming it as a test parameter, and its return value is passed in.
- Teardown is cleanup after a test. A yield fixture puts set-up and teardown in one function.
- Code after the `yield` runs even when the test fails, so use it to call `close()`.
