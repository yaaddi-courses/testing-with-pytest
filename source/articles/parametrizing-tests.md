# Parametrizing tests and fixtures

**Covers:** parametrize, parametrized fixture, request.param

## One test, many cases {#overview}
A function usually has to work for many inputs: the typical ones and the edges. Writing one test per input repeats the same three lines over and over. A loop inside a test is no better: the first failure ends the test, and you do not see the others. pytest offers a cleaner approach, which turns one test function into many separate tests.

## parametrize {#parametrize}
`@pytest.mark.parametrize` is a decorator that runs a test function once for each set of values that you provide. You name the arguments and give a list of cases:

```text
@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

pytest creates three separate tests, shown as `test_add[1-2-3]`, `test_add[0-0-0]` and `test_add[-1-1-0]`. Each passes or fails by itself, so one failing case does not hide the others, and you can run a single case by its name. You can supply readable names with `ids=[...]`, and you can mark one case with `pytest.param(..., marks=...)` if one case needs special treatment.

Stacking two parametrize decorators on one test produces every combination, called the cross product. Three values times four values gives twelve tests.

## parametrized fixture {#parametrized-fixture}
A parametrized fixture is a fixture created with `@pytest.fixture(params=[...])`. pytest builds one instance of the fixture for each value in the list, and runs every test that uses it once for each. Use it when the thing that varies is part of the set-up, such as which database or which settings object the tests run against, and many tests should run against each variation.

```text
@pytest.fixture(params=["sqlite", "postgres"])
def database(request):
    return connect(request.param)

def test_insert(database):
    ...
```

Here every test that uses `database` runs twice, once for each backend, without any change to the tests.

## request.param {#request-param}
Inside a parametrized fixture, `request.param` holds whichever single value from the `params` list is being used for the current run. The fixture asks for the special `request` parameter, and reads `.param` from it. The fixture then builds what it needs from the value, as in `connect(request.param)` above.

## A worked example {#example}
This program shows the idea of parametrization and counts the tests it would create.

```python
def add(a, b):
    return a + b


cases = [(1, 2, 3), (0, 0, 0), (-1, 1, 0)]
for a, b, expected in cases:
    assert add(a, b) == expected
print(len(cases), "cases passed")
# 3 cases passed
```

How to read it: the loop shows the data-driven idea, with each row holding the inputs and the expected result. In pytest, the same table becomes three tests and a report line for each.

Now the count for stacking and for fixtures:

| Setup | Tests created |
|---|---|
| `parametrize` with 3 rows | 3 |
| Two stacked `parametrize` decorators with 3 and 4 values | 3 x 4 = 12 |
| One test using a fixture with 2 params | 2 |
| 10 tests using that same fixture | 10 x 2 = 20 |

The last row is the strength of the fixture version. With ten tests that must pass on both databases, you write ten tests and one fixture and pytest runs 20 cases. Without it, you would copy each test, which is 20 functions, and keep them in step.

Compare the loop with the parametrized version when a case fails. With the loop, if the second row is wrong, the test stops there, and you never learn about the third. With `parametrize`, the report says, for example, `2 passed, 1 failed`, and names `test_add[0-0-1]` as the failure, and the other cases still ran.

## A common mistake {#mistake}
Putting too many cases, or cases that test different behaviours, into one parametrize table, so the test body fills with `if` statements. Each table should test one behaviour. Another mistake is unreadable ids. Give them names when the values are long or are objects.

## Key points {#key-points}
- `@pytest.mark.parametrize` turns one test into one test per row, each reported separately.
- Stacked decorators multiply: 3 times 4 gives 12 tests.
- A parametrized fixture runs every test that uses it once per value in `params`.
- Inside such a fixture, `request.param` is the current value.
