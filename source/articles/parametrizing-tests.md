# Parametrizing tests and fixtures

**Covers:** parametrize, parametrized fixture, request.param

## One test, many cases {#overview}
A function usually has to work for many inputs, both the typical ones and the edges. If you write one test per input, you repeat the same three lines again and again. A loop inside a single test is not much better, because the first failure ends the test and you never see the others. pytest has a cleaner way: it turns one test function into many separate tests.

## parametrize {#parametrize}
`@pytest.mark.parametrize` is a decorator that runs a test function once for each set of values you give it. You name the arguments and then list the cases:

```text
@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

pytest creates three separate tests, shown as `test_add[1-2-3]`, `test_add[0-0-0]` and `test_add[-1-1-0]`. Each one passes or fails on its own, so a failing case does not hide the others, and you can run a single case by its name. You can supply readable names with `ids=[...]`, and if one case needs special treatment, you can mark it with `pytest.param(..., marks=...)`.

If you stack two parametrize decorators on one test, you get every combination, which is called the cross product. Three values times four values gives twelve tests.

## parametrized fixture {#parametrized-fixture}
Sometimes the thing that varies is not an input but the set-up: which database, which settings object. A parametrized fixture handles that. You create it with `@pytest.fixture(params=[...])`, and pytest builds one instance for each value in the list and runs every test that uses the fixture once for each.

```text
@pytest.fixture(params=["sqlite", "postgres"])
def database(request):
    return connect(request.param)

def test_insert(database):
    ...
```

Every test that uses `database` now runs twice, once for each backend, and you do not change the tests.

## request.param {#request-param}
Inside a parametrized fixture, `request.param` holds the single value from the `params` list that is in use for the current run. The fixture asks for the special `request` parameter and reads `.param` from it. Then it builds what it needs from that value, as `connect(request.param)` does above.

## A worked example {#example}
This program shows the idea of parametrization and counts the cases.

```python
def add(a, b):
    return a + b


cases = [(1, 2, 3), (0, 0, 0), (-1, 1, 0)]
for a, b, expected in cases:
    assert add(a, b) == expected
print(len(cases), "cases passed")
# 3 cases passed
```

The loop shows the data-driven idea: each row holds the inputs and the expected result. In pytest, the same table becomes three tests, with a report line for each.

Now let's count the tests for stacking and for fixtures:

| Setup | Tests created |
|---|---|
| `parametrize` with 3 rows | 3 |
| Two stacked `parametrize` decorators with 3 and 4 values | 3 × 4 = 12 |
| One test using a fixture with 2 params | 2 |
| 10 tests using that same fixture | 10 × 2 = 20 |

The last row shows the strength of the fixture version. If ten tests must pass on both databases, you write ten tests and one fixture, and pytest runs 20 cases. Without the fixture, you would copy each test, which gives 20 functions to keep in step.

Compare the loop with the parametrized version when a case fails. In the loop, if the second row is wrong, the test stops there and you never learn about the third. With `parametrize`, the report says, for example, `2 passed, 1 failed`, and it names `test_add[0-0-1]` as the failure, while the other cases still ran.

## A common mistake {#mistake}
A common slip is to put too many cases, or cases that test different behaviours, into one parametrize table, so that the test body fills up with `if` statements. Each table should test one behaviour. Another slip is unreadable ids. Give the cases names when the values are long or are objects.

## Key points {#key-points}
- `@pytest.mark.parametrize` turns one test into one test per row, each reported separately.
- Stacked decorators multiply: 3 times 4 gives 12 tests.
- A parametrized fixture runs every test that uses it once per value in `params`.
- Inside such a fixture, `request.param` is the current value.
