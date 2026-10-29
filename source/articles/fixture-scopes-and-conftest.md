# Fixture scopes, test isolation and conftest.py

**Covers:** fixture scope, session scope, class scope, test isolation, conftest.py

## How long should a prepared thing live? {#overview}
By default pytest rebuilds a fixture for every test. That is safe, but it can be slow. If the fixture starts a database or loads a large file, doing it a thousand times wastes minutes. So pytest lets you choose how long a fixture lives, which is a trade between speed and safety. It also has a standard place for fixtures that many files share. Let's start with the safety side.

## test isolation {#test-isolation}
Test isolation means that one test's set-up, data and side effects cannot leak into another test. Isolated tests can run in any order, alone or together, and give the same result. Without isolation, a test passes or fails depending on what ran before it, and the failures are hard to reproduce.

Weak isolation is easy to get away with at first. Two tests share a list, and it works in the order they happen to run. Then someone adds a test, or runs the suite in parallel, and a mysterious failure appears. Judge every scope choice below by how it affects isolation.

## fixture scope {#fixture-scope}
A fixture's scope is the `scope=` setting that decides how often pytest rebuilds it. From the narrowest to the widest, the levels are:

- `function` (the default): rebuilt for every test.
- `class`: once for each test class.
- `module`: once for each test file.
- `package`: once for each package of tests.
- `session`: once for the whole run.

A wider scope is faster, because the fixture is built less often, but more tests share it, so isolation is weaker. A narrow scope is slow but safe. The usual advice is to keep the default unless building the fixture is expensive, and then to widen the scope only for fixtures that tests read but do not change.

```text
@pytest.fixture(scope="session")
def database():
    ...
```

## class scope {#class-scope}
With class scope, pytest builds the fixture once for each test class. All the test methods in the class share it, and pytest rebuilds it when it moves on to the next class. It suits a group of related tests that read the same prepared data, for example all the tests in a class that exercise one object.

## session scope {#session-scope}
With session scope, pytest builds the fixture exactly once for the entire run, and every test that asks for it reuses it. This is the fastest option and the least isolated one. Typical uses are starting a test server, creating a database schema or loading a large reference file. Tests should not change anything built at session level, or a narrower fixture should reset it after each test.

## conftest.py {#conftest-py}
conftest.py is a file with a special name that pytest loads automatically. Any fixture defined in it becomes available to every test in that directory and below, and you import nothing. You put the shared fixtures there, and tests ask for them by name.

A project can have several. One at the top holds fixtures used everywhere, and others in sub-folders hold local ones. A conftest.py that is closer to the test can override a fixture of the same name from a higher one. It can also hold plugins and hooks. Do not import from `conftest.py` in your tests, because pytest does that work for you.

## A worked example {#example}
This program counts how often an expensive fixture would be built for five tests.

```python
builds = 0


def make_db():
    global builds
    builds += 1
    return {"id": builds}


for _ in range(5):
    make_db()
print(builds)

builds = 0
db = make_db()
for _ in range(5):
    assert db["id"] == 1
print(builds)
# 5
# 1
```

Let's read it.

1. With function scope, each of the five tests gets its own database, so there are 5 builds.
2. With session scope, the database is built once and the five tests share it, so there is 1 build.

Now put numbers on the speed. If building the database takes 2 seconds and the suite has 300 tests, function scope costs 300 × 2 = 600 seconds, or 10 minutes. Session scope costs 2 seconds. That is the reason to widen the scope of expensive things.

The price is isolation. If one test inserts a row into the shared database and another counts the rows, the count depends on the order of the tests. A common fix is to keep the database at session scope and add a function-scoped fixture that starts a transaction for each test and rolls it back afterwards, so that changes never stay.

Where does it live? The expensive `database` fixture goes in `tests/conftest.py`, and every test file below it can use it by naming `database`. A file `tests/api/conftest.py` can add a fixture that only the API tests need.

## A common mistake {#mistake}
A common slip is to widen a scope to make the tests faster, and then to have tests that change the shared object, so the results depend on the order. Widen only what is read-only, or reset it. Another slip is importing fixtures from `conftest.py` by hand. pytest provides them automatically.

## Key points {#key-points}
- Test isolation means that tests do not affect each other.
- Scope decides how often a fixture is built: function, class, module, package or session.
- A wider scope is faster and less isolated. Widen only read-only or resettable fixtures.
- `conftest.py` shares fixtures with all the tests below it, with no import.
