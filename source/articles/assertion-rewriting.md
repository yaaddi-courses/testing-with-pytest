# Assertion rewriting and readable failures

**Covers:** assertion rewriting, asserting

## Why a plain assert can give a rich message {#overview}
You have probably seen what Python's ordinary `assert` does when it fails: a bare `AssertionError`, and nothing about the values it compared. That is not much help when you are trying to find a bug. pytest fills the gap, and you do not change how you write the test. Let's look at how it does that, because knowing the mechanism clears up a few surprises.

## asserting {#asserting}
Asserting means stating, inside a test, that a condition must be true, so that the test fails if it is not. A test usually has one main thing to assert and sometimes a few related ones. The more specific the assertion, the more it tells you when it fails: `assert len(items) == 3` is better than `assert items`, which only says that the list is not empty.

A useful habit is to assert on behaviour that a user would notice: the returned value, the new state or the message shown. If you assert on internal details, such as how many times a private method ran, the test breaks whenever the code is improved, even though nothing the user sees has changed.

## assertion rewriting {#assertion-rewriting}
Where do the values in a failure message come from? pytest puts them there. As it loads your test file, it reads the syntax tree of the module and replaces each plain assert with code that remembers the intermediate results. That technique is assertion rewriting.

You write `assert total == expected`. When it fails, pytest prints something like:

```text
    assert total == expected
E   assert 98 == 100
E    +  where 98 = compute_total(items)
```

You see the actual and expected values, and the call that produced one of them. For longer structures, such as two lists or two dictionaries, pytest shows exactly which element differs. For strings, it shows a character diff.

One limit is worth knowing. The rewriting applies to test files, to the shared helper files that pytest loads itself and to any module you register with `pytest.register_assert_rewrite`. Ordinary library code is not rewritten, so an assert inside a plain helper module shows only the ordinary message. If your assertions live in a helper in another module, register that module, or have the helper return a boolean and assert on it in the test.

## A worked example {#example}
We will see what an assertion needs to say, and how you would add a message by hand when the automatic one is not enough.

```python
def compute_total(prices, discount):
    return sum(prices) - discount


prices = [20, 30, 50]
total = compute_total(prices, 2)
expected = 100
message = f"{total} != {expected} (prices={prices})"
print(message)
# 98 != 100 (prices=[20, 30, 50])
```

Let's read it step by step.

1. The function subtracts a discount of 2 from the sum of the prices, which is 100, and gives 98.
2. The test expects 100, so `assert total == expected` would fail.
3. With assertion rewriting, pytest shows `98 == 100` and the call `compute_total(prices, 2)`, and you write no extra code.
4. The f-string above builds the kind of message you would have to write by hand in plain Python. Rewriting does that work for you.

Now compare three ways of asserting on a list of results:

| Style | What a failure tells you |
|---|---|
| `assert results` | Nothing, only that it was empty |
| `assert results == ["a", "b"]` | The two lists, and the first position where they differ |
| `assert "a" in results and "b" in results` | Which half failed is not shown unless it is split into two assertions |

In the second style, if `results` is `["a", "c"]`, pytest points to index 1, where it found `"c"` and expected `"b"`. You do not need to add a print and run the test again.

Sometimes the check has to live in another file. You can then write a helper that does the comparison and returns `True` or `False`, and the test does `assert is_valid(order)`. The failure shows `assert False` plus the call, which tells you less than comparing the values directly in the test.

## A common mistake {#mistake}
A common slip is to put several unrelated checks into one assert with `and`, or to wrap everything in `try` and `except AssertionError`, which hides the failure. Write one clear assertion for each behaviour. Another slip is to expect a rich message from an assert inside a library module that is not registered for rewriting.

## Key points {#key-points}
- Asserting means checking a condition in a test. Be specific about what you assert.
- Assertion rewriting makes pytest show the values behind a failed assert, and you write no code for it.
- Rewriting covers test files, pytest's shared helper files and registered modules.
- Compare values directly in the test to get the best failure messages.
