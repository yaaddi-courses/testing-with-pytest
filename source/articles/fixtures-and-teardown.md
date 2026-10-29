# Fixtures: reusable setup and teardown

**Covers:** fixture, decorator, teardown, close(), yield fixture

## Setting the stage for a test {#overview}
Most tests need something prepared first: a user record, a temporary file, an open connection. Writing that preparation at the top of every test repeats code and hides what each test is really about. pytest solves it with fixtures, small functions that prepare things and hand them to any test that asks. This article covers how they are declared, how they are used and how they clean up.

## decorator {#decorator}
A decorator is a Python feature, written as `@something` directly above a function definition, which wraps or registers that function. For example, `@pytest.fixture` above a function tells pytest that the function is a fixture. The line above `def` is an ordinary Python expression that receives the function and returns a version of it, with extra behaviour or a record in a registry. You will meet decorators throughout pytest, in fixtures, labels and parametrization.

## fixture {#fixture}
A fixture is a reusable set-up function, marked with the `@pytest.fixture` decorator, that a test can use by naming it as a parameter. pytest sees the parameter name, finds the fixture with that name, runs it and passes its return value to the test.

```text
import pytest

@pytest.fixture
def user():
    return {"name": "Ana", "role": "admin"}

def test_role(user):
    assert user["role"] == "admin"
```

The test does not call `user()`. It asks for it, which is called dependency injection. A fixture can use other fixtures in the same way, so set-up is built from small, named pieces. By default, a fixture is run again for each test, so every test gets a fresh object.

## close() {#close}
`close()` is the method that releases a file, a network connection or a database handle when you have finished with it. Forgetting it leaks resources: files stay locked, connections run out and data may not be written. A fixture that opens such a resource should also arrange for `close()` to be called afterwards, even if the test fails.

## teardown {#teardown}
Teardown is the cleanup code that a fixture runs after a test is done with it: closing a file, deleting a temporary record, stopping a server. It restores the world to how it was, so that one test's leftovers cannot disturb the next. The word comes from the set-up and teardown pairs of the older xUnit frameworks.

## yield fixture {#yield-fixture}
A yield fixture is a fixture that uses `yield` instead of `return`. The code before the `yield` is the set-up, the yielded value goes to the test and the code after the `yield` is the teardown.

```text
@pytest.fixture
def connection():
    conn = open_connection()
    yield conn
    conn.close()
```

pytest runs the part after the yield when the test finishes, whether it passed or failed. That is the main benefit: the cleanup sits right beside the set-up, in the same function, and cannot be skipped by a failing assertion in the test.

If the set-up itself fails before the `yield`, the cleanup does not run, so a fixture that creates several things should be built so that each has its own cleanup, or use fixtures that depend on one another.

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

How to read it:

1. `next(gen)` runs the function up to the `yield`. That prints "setup" and produces the value `"connection"`.
2. The test would run at this point, with the value in hand. Here we just print it.
3. The second `next(gen)` resumes the function after the `yield`. It prints "teardown", the function ends and `StopIteration` is raised, which we ignore.

That is what pytest does around a test: ask for the value, run the test and resume the fixture so that it can clean up.

Compare the two styles for a test that needs a file:

| | In every test | As a yield fixture |
|---|---|---|
| Lines of set-up and cleanup per test | About 4 | 0 |
| Cleanup if the assertion fails | Skipped unless wrapped in `try` and `finally` | Always runs |
| For 30 tests that need the file | 120 repeated lines | One 5-line fixture |

With the fixture, the thirty tests each hold only the assertion that matters, and a fix to the cleanup is made once.

## A common mistake {#mistake}
Calling a fixture function directly, as `user()`, instead of asking for it as a parameter. pytest forbids it. Another mistake is cleanup written after an assertion in the test itself. If the assertion fails, the cleanup never runs. Put it in the fixture, after the yield.

## Key points {#key-points}
- A decorator such as `@pytest.fixture` registers or wraps the function below it.
- A fixture is requested by naming it as a test parameter, and its return value is passed in.
- Teardown is cleanup after a test. A yield fixture puts set-up and teardown in one function.
- Code after the `yield` runs even when the test fails, so use it to call `close()`.
