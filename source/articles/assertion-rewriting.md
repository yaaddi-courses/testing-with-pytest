# Assertion rewriting and readable failures

**Covers:** assertion rewriting, asserting

## Why a plain assert can give a rich message {#overview}
Python's ordinary assert statement is very short on information. When it fails, you get a bare `AssertionError`, and nothing about the values that were compared. pytest adds the information, with no change to how you write the test. The mechanism is neat, and understanding it clears up a few surprises.

## asserting {#asserting}
Asserting means checking in a test that a condition is true, and failing the test if it is not. A test usually has one main thing to assert, and often several related ones. Good assertions are specific: `assert len(items) == 3` is better than `assert items`, which only says that the list is not empty.

A useful rule is to assert on behaviour that a user would notice: the returned value, the new state or the message shown. Asserting on internal details, such as how many times a private method ran, makes a test break when the code is improved.

## assertion rewriting {#assertion-rewriting}
Assertion rewriting is pytest's technique of changing your plain assert statements, as the test file is loaded, so that when one fails, it reports the values that were involved. pytest reads the test module's syntax tree and replaces each assert with code that remembers the intermediate results.

You write `assert total == expected`. When it fails, pytest prints something like:

```text
    assert total == expected
E   assert 98 == 100
E    +  where 98 = compute_total(items)
```

It shows the actual and expected values, and the call that produced one of them. For longer structures, such as two lists or two dictionaries, it shows exactly which element differs. For strings, it shows a diff of the characters.

The rewriting applies to test files, to the shared helper files that pytest loads itself and to any other modules that you register with `pytest.register_assert_rewrite`. Plain library code is not rewritten, so an assert inside an ordinary helper module shows only the ordinary message. If your assertions live in a helper function in another module, register that module, or return a boolean and assert on it in the test.

## A worked example {#example}
The following shows what an assertion needs to say, and how a message can be added by hand when the automatic one is not enough.

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

How to read it:

1. The function subtracts a discount of 2 from the sum of 100, giving 98.
2. The test expects 100, so `assert total == expected` would fail.
3. With assertion rewriting, pytest shows `98 == 100` and the call `compute_total(prices, 2)` without extra code from you.
4. The f-string above builds the kind of message that you would add manually in plain Python, and that is what rewriting automates.

Compare two ways of asserting on a list of results:

| Style | What a failure tells you |
|---|---|
| `assert results` | Nothing, only that it was empty |
| `assert results == ["a", "b"]` | The two lists, and the first position where they differ |
| `assert "a" in results and "b" in results` | Which half failed is not shown unless it is split into two assertions |

For the second style, if `results` is `["a", "c"]`, pytest points to index 1, where `"c"` was found and `"b"` expected. That saves you from adding a print and rerunning.

If the check must live in another file, an alternative is a helper that does the comparison and returns `True` or `False`, and the test does `assert is_valid(order)`. The failure then shows `assert False` plus the call, which is less informative than comparing the values directly in the test.

## A common mistake {#mistake}
Putting several unrelated checks in one assert with `and`, or wrapping the whole thing in `try` and `except AssertionError`, which hides the failure. Write one clear assertion per behaviour. Another mistake is expecting a rich message from an assert in a library module that is not registered for rewriting.

## Key points {#key-points}
- Asserting means checking a condition in a test. Be specific about what you assert.
- Assertion rewriting makes pytest show the values behind a failed assert, with no code from you.
- Rewriting covers test files, pytest's shared helper files and registered modules.
- Compare values directly in the test for the best failure messages.
