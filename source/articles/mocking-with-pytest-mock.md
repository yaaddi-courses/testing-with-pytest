# Mocking with pytest-mock

**Covers:** pytest-mock, mocker, return_value, side_effect, assert_called_with, assert_called_once, API, HTTP

## Keeping tests away from the outside world {#overview}
Code often depends on things that a test should not touch: a web service that costs money or may be down, a clock, a payment system. If a test calls them for real, it becomes slow, unreliable and sometimes harmful. The usual answer is to replace the dependency with a stand-in for the duration of the test. Python has the tools built in, and a plugin makes them tidier to use in pytest.

## HTTP {#http}
HTTP, or HyperText Transfer Protocol, is the protocol that web browsers and servers use to exchange pages and data. Most programs that "call an API" do so over HTTP: they send a request to a web address and read the answer. These calls are the most common reason to use mocks, because the real server is outside your control.

## API {#api}
An API, or application programming interface, is a defined way for one program to ask another for something. A weather service offers an API: send a city, get a forecast. In a test, you want to check your own code's reaction to the answer without calling the real service, so you replace the function or object that makes the call.

## pytest-mock {#pytest-mock}
pytest-mock is a plugin, installed separately with `pip install pytest-mock`, that adds the `mocker` fixture. It is a thin, convenient layer over the standard library's `unittest.mock`. The advantage is that everything you patch is undone automatically when the test ends, and you do not need `with` blocks or decorators.

## mocker {#mocker}
`mocker` is the fixture that pytest-mock provides for patching things during a test. The most common call is `mocker.patch("module.function")`, which replaces that name with a mock object while the test runs. It also offers `mocker.Mock()`, `mocker.spy()` and others. After the test, the original is restored.

An important rule: patch the name where it is looked up, and not where it was defined. If `shop.py` does `from client import fetch_rate`, patch `shop.fetch_rate`, since that is the name that `shop` actually uses.

## return_value {#return-value}
`return_value` fixes what a mock hands back every time it is called. `mocker.patch("shop.fetch_rate", return_value=1.25)`, or later `mock.return_value = 1.25`. It lets you control the input that the code under test receives, so that the result is predictable.

## side_effect {#side-effect}
`side_effect` controls more complex behaviour than a fixed return value. It can be an exception, which the mock raises when called, a function, which is called with the same arguments and whose result is returned, or a list, from which successive calls take successive values. Use an exception to test how your code copes with a failure, such as a timeout.

## assert_called_with {#assert-called-with}
`mock.assert_called_with(...)` checks that the mock was called, and that the most recent call used exactly the arguments you expect. If the arguments differ, the test fails and shows both. Variants check all calls (`assert_has_calls`) or that no call happened (`assert_not_called`).

## assert_called_once {#assert-called-once}
`mock.assert_called_once()` checks that the mock was called exactly once, with any arguments. A related method, `assert_called_once_with(...)`, checks the single call and its arguments together. It catches a bug where code calls a service twice, for example by retrying when it should not.

## A worked example {#example}
This program uses the standard-library tools that `mocker` wraps.

```python
from unittest import mock


def fetch_rate(client):
    return client.get("/rate")["usd"]


client = mock.Mock()
client.get.return_value = {"usd": 1.25}
print(fetch_rate(client))
client.get.assert_called_once_with("/rate")
client.get.side_effect = TimeoutError("slow")
try:
    fetch_rate(client)
except TimeoutError as error:
    print(error)
# 1.25
# slow
```

How to read it:

1. The mock `client` takes the place of a real HTTP client. Its `get` method returns a prepared answer, 1.25, through `return_value`.
2. `fetch_rate` gets the answer and extracts the number, so the first print shows 1.25.
3. `assert_called_once_with("/rate")` confirms that the function asked for the right address, once. If it had called the service twice, this line would fail.
4. Setting `side_effect` to a `TimeoutError` makes the next call fail, which tests the failure path without any real network trouble.

The pytest version of one of these tests:

```text
def test_price_in_eur(mocker):
    mocker.patch("shop.fetch_rate", return_value=0.9)
    assert price_in_eur(100) == 90
```

Count what the mock saves: a real call takes about 300 ms and fails sometimes. With 50 such tests, that is 15 seconds of waiting and a handful of random failures a month. With mocks, the 50 tests take a few milliseconds and never fail because of the network.

The cost is that a mock only proves that your code uses the service as you think it does. Keep a few tests that call the real thing, in a separate, slower group.

## A common mistake {#mistake}
Patching the wrong name, so the real function is still called and the mock is ignored. Patch where it is used. Another mistake is mocking so much that the test just repeats the implementation: if every line of the code is mirrored by an `assert_called_with`, a harmless refactoring breaks the test.

## Key points {#key-points}
- Mocks replace slow or unreliable dependencies, such as HTTP calls to an API.
- The `mocker` fixture from pytest-mock patches things and undoes it automatically.
- `return_value` fixes the answer and `side_effect` raises errors or varies results.
- `assert_called_with` and `assert_called_once` check how the mock was used. Patch where the name is used.
