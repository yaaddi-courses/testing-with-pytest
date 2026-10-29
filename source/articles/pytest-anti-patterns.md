# Common pytest anti-patterns and better habits

**Covers:** anti-pattern

## Habits that look fine until they do not {#overview}
Most test suites do not fail in one big way. They get gradually worse: slower, more confusing, harder to trust. The causes are often habits that looked reasonable when they were written. Recognising the common ones early saves a lot of pain later.

## anti-pattern {#anti-pattern}
An anti-pattern is a habit or structure that looks reasonable on the surface but reliably causes problems later. In testing, the examples below show how a shortcut that saves a minute today costs an hour next month. The term is useful because it gives a name to the thing, which makes it easier to spot in a code review.

## Tests that depend on order {#order}
A test that passes only because another one ran first is fragile. It breaks when tests are run alone, in a different order or in parallel. It usually comes from shared mutable state, such as a module-level list or a database row left behind. Give every test its own data, through a fixture, and reset anything shared.

## Over-broad fixtures {#fixtures}
A fixture that sets up everything, "just in case", with `autouse=True` and a wide scope, makes every test slow and hides what each one needs. A reader cannot tell which test depends on which set-up. Make fixtures small and ask for them by name. Use `autouse` only for things that truly apply to every test, such as a clean environment.

## Testing implementation, not behaviour {#implementation}
Tests that check how the code works inside, such as which private method was called or how many times, break whenever the code is improved, even if nothing changed for the user. Test what goes in and what comes out. A good test survives a refactoring.

## Over-mocking {#over-mocking}
When everything is mocked, the test only checks that the code calls the mocks the way it is written to. It can pass while the real system is broken. Mock only what is slow, unreliable or outside your control, and keep some tests that use real parts together.

## Sleeping and guessing {#sleeping}
Using `time.sleep(2)` to wait for something makes the suite slow and still unreliable, since two seconds may not be enough on a busy machine. Wait for a condition with a timeout instead, or make the code testable without waiting by passing in a fake clock.

## Vague assertions {#vague}
`assert result` passes for anything that is truthy. `assert len(items) > 0` hides the real number. Be exact: `assert items == ["a", "b"]`. A precise assertion is also much easier to debug when it fails.

## A worked example {#example}
A test pair that depends on order:

```python
shared = []


def test_one():
    shared.append("a")
    assert shared == ["a"]


def test_two():
    shared.append("b")
    assert shared == ["b"]


test_one()
try:
    test_two()
except AssertionError:
    print("test_two fails when it runs after test_one")
# test_two fails when it runs after test_one
```

How to read it:

1. Both tests use a module-level list. `test_one` adds "a" and expects exactly `["a"]`.
2. `test_two` adds "b" and expects exactly `["b"]`. Run alone, it passes. Run after `test_one`, the list is `["a", "b"]`, so it fails.
3. The result depends on the order, so `pytest -k two` passes and the full run fails. That kind of difference is the sign of this anti-pattern.

The fix is a fixture that returns a new list for each test:

```text
@pytest.fixture
def shared():
    return []

def test_one(shared):
    shared.append("a")
    assert shared == ["a"]
```

Count the effect on debugging time. Suppose an order-dependent failure appears once a week, and each time it takes 45 minutes to find the cause, because nobody can reproduce it alone. That is 45 x 52 = 2,340 minutes a year, about 39 hours. A fixture that costs 2 lines per test removes it.

A short checklist for review: Does the test pass alone, in a random order and in parallel? Does it check behaviour that a user would notice? Does it say exactly what it expects? If the answer to any is no, fix it before it spreads.

## A common mistake {#mistake}
Fixing a flaky or order-dependent test by adding a sleep or by re-running it. That hides the cause, and it will return. Find the shared state and remove it. Another mistake is following a rule mechanically, for instance avoiding all mocks, when the real answer is to choose with a reason.

## Key points {#key-points}
- An anti-pattern looks fine and causes trouble later. Learn to name it.
- Avoid order dependence, over-broad fixtures, testing internals, over-mocking, sleeping and vague assertions.
- Check that each test passes alone, in a different order and in parallel.
- Fix the cause, not the symptom, and do not hide flakiness with retries.
