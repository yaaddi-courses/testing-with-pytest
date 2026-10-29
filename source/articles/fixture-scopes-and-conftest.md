# Fixture scopes, test isolation and conftest.py

**Covers:** fixture scope, session scope, class scope, test isolation, conftest.py

## How long should a prepared thing live? {#overview}
By default a fixture is rebuilt for every test, which is safe but can be slow. If the fixture starts a database or loads a large file, doing so a thousand times wastes minutes. pytest lets you choose how long a fixture lives, which trades speed against safety. It also has a standard place to put fixtures that many files share.

## test isolation {#test-isolation}
Test isolation is the property that one test's set-up, data and side effects cannot leak into another test. Isolated tests can run in any order, alone or together, and give the same result. Without it, a test passes or fails depending on what ran before it, and failures are hard to reproduce.

Weak isolation is easy to get away with at first. Two tests share a list, and it happens to work in the order they run in. Then someone adds a test, or runs them in parallel, and a mysterious failure appears. Every scope choice below should be judged by how it affects isolation.

## fixture scope {#fixture-scope}
A fixture's scope is the setting, `scope=`, that decides how often pytest rebuilds it. The levels, from the narrowest to the widest, are:

- `function` (the default): rebuilt for every test.
- `class`: once for each test class.
- `module`: once for each test file.
- `package`: once for each package of tests.
- `session`: once for the whole run.

A wider scope is faster, since the fixture is built less often, but it is shared between more tests, so isolation is weaker. A narrow scope is slow but safe. The usual guidance is to use the default unless building the fixture is expensive, and then widen it only for fixtures that are read-only or that the tests do not change.

```text
@pytest.fixture(scope="session")
def database():
    ...
```

## class scope {#class-scope}
Class scope means that a fixture is built once for each test class and shared by all test methods in it, then rebuilt when pytest moves to the next class. It suits a group of related tests that read from the same prepared data, for example all the tests in a class that exercise a single object.

## session scope {#session-scope}
Session scope means that a fixture is built exactly once for the entire pytest run and reused by every test that asks for it. It is the fastest and the least isolated. Typical uses are starting a test server, creating a database schema or loading a large reference file. Anything built at session level should not be changed by tests, or should be reset by a narrower fixture after each one.

## conftest.py {#conftest-py}
conftest.py is a specially named file that pytest loads automatically. Any fixture defined in it becomes available to every test in that directory and below, with no import. Put shared fixtures there, and tests just ask for them by name.

A project can have several, one at the top for fixtures used everywhere and others in sub-folders for local ones. A closer conftest.py can override a fixture of the same name from a higher one. It can also hold plugins and hooks. Do not import from `conftest.py` in your tests; let pytest do the work.

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

How to read it:

1. With function scope, each of the five tests gets its own database: 5 builds.
2. With session scope, the database is built once and shared by the five tests: 1 build.

Now put numbers on the speed. If building the database takes 2 seconds and the suite has 300 tests, function scope costs 300 x 2 = 600 seconds, or 10 minutes. Session scope costs 2 seconds. That is the reason to widen scope for expensive things.

The price is isolation. If one test inserts a row into the shared database and another counts the rows, the count depends on the order. A common fix is to keep the database at session scope but give each test a function-scoped fixture that starts a transaction and rolls it back afterwards, so that changes never remain.

Where to put it: the expensive `database` fixture goes in `tests/conftest.py`. Every test file below can then use it just by naming `database`. A file `tests/api/conftest.py` can add a fixture that only the API tests need.

## A common mistake {#mistake}
Widening the scope to make tests faster, and then having tests that change the shared object, so that results depend on the order. Widen only what is read-only, or reset it. Another mistake is importing fixtures from `conftest.py` by hand. pytest provides them automatically.

## Key points {#key-points}
- Test isolation means that tests do not affect each other.
- Scope decides how often a fixture is built: function, class, module, package or session.
- A wider scope is faster and less isolated. Widen only read-only or resettable fixtures.
- `conftest.py` shares fixtures with all the tests below it, with no import.
