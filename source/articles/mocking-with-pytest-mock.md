# Mocking with pytest-mock

**Covers:** pytest-mock, mocker, return_value, side_effect, assert_called_with, assert_called_once, API, HTTP

## Keeping tests away from the outside world {#overview}
Imagine your code asks a web service for today's exchange rate. If your test makes that real call, it becomes slow, it fails whenever the service is down, and it may even cost money. The usual answer is to replace the dependency with a stand-in for the length of the test. Python has the tools for this built in, and a plugin makes them tidier to use in pytest. Let's start with the two terms that explain why you need it.

## HTTP {#http}
HTTP, the HyperText Transfer Protocol, is the language that web browsers and servers use to exchange pages and data. Most programs that "call an API" do it over HTTP: they send a request to a web address and read the answer. These calls are the most common reason to use mocks, because the real server is outside your control.

## API {#api}
An API, or application programming interface, is an agreed way for one program to ask another for something. A weather service offers an API: you send a city and you get a forecast. In a test, you want to see how your own code reacts to the answer, without calling the real service, so you replace the function or object that makes the call.

## pytest-mock {#pytest-mock}
pytest-mock is a plugin you install separately with `pip install pytest-mock`, and it adds the `mocker` fixture. It is a thin, convenient layer over `unittest.mock` from the standard library. Its advantage is that everything you patch is undone automatically when the test ends, so you need no `with` blocks or decorators.

## mocker {#mocker}
`mocker` is the fixture that pytest-mock provides for patching things during a test. The most common call is `mocker.patch("module.function")`, which replaces that name with a mock object while the test runs. It also offers `mocker.Mock()`, `mocker.spy()` and more. After the test, the original comes back.

There is one rule to learn early: patch the name where it is looked up, not where it was defined. If `shop.py` does `from client import fetch_rate`, patch `shop.fetch_rate`, because that is the name `shop` actually uses.

## return_value {#return-value}
`return_value` fixes what a mock hands back each time it is called. You write `mocker.patch("shop.fetch_rate", return_value=1.25)`, or set `mock.return_value = 1.25` later. This lets you control the input that the code under test receives, so that the result is predictable.

## side_effect {#side-effect}
`side_effect` covers behaviour that a fixed return value cannot. It can be an exception, which the mock raises when called. It can be a function, which is called with the same arguments and whose result is returned. Or it can be a list, from which successive calls take successive values. An exception is the way to test how your code copes with a failure, such as a timeout.

## assert_called_with {#assert-called-with}
`mock.assert_called_with(...)` checks that the mock was called, and that the most recent call used exactly the arguments you expect. If the arguments differ, the test fails and shows both sets. Related variants check all the calls (`assert_has_calls`) or that nothing was called (`assert_not_called`).

## assert_called_once {#assert-called-once}
`mock.assert_called_once()` checks that the mock was called exactly once, whatever the arguments. A related method, `assert_called_once_with(...)`, checks the single call and its arguments together. It catches the bug where code calls a service twice, for example by retrying when it should not.

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

Let's go through it.

1. The mock `client` takes the place of a real HTTP client. Its `get` method returns a prepared answer, 1.25, through `return_value`.
2. `fetch_rate` receives the answer and extracts the number, so the first print shows 1.25.
3. `assert_called_once_with("/rate")` confirms that the function asked for the right address, once. If it had called the service twice, this line would fail.
4. Setting `side_effect` to a `TimeoutError` makes the next call fail. That tests the failure path without any real network trouble.

Here is the pytest version of one such test:

```text
def test_price_in_eur(mocker):
    mocker.patch("shop.fetch_rate", return_value=0.9)
    assert price_in_eur(100) == 90
```

Count what the mock saves. A real call takes about 300 ms and sometimes fails. With 50 such tests, that is 50 × 300 ms = 15 seconds of waiting and a handful of random failures a month. With mocks, the 50 tests take a few milliseconds and never fail because of the network.

There is a price. A mock only proves that your code uses the service the way you think it does. Keep a few tests that call the real thing, in a separate, slower group.

## A common mistake {#mistake}
A common slip is to patch the wrong name, so the real function still runs and the mock is ignored. Patch where the name is used. Another slip is to mock so much that the test only repeats the implementation. If every line of the code is mirrored by an `assert_called_with`, a harmless refactoring breaks the test.

## Key points {#key-points}
- Mocks replace slow or unreliable dependencies, such as HTTP calls to an API.
- The `mocker` fixture from pytest-mock patches things and undoes it automatically.
- `return_value` fixes the answer, and `side_effect` raises errors or varies results.
- `assert_called_with` and `assert_called_once` check how the mock was used. Patch where the name is used.
