# Custom markers and strict registration

**Covers:** marker, strict-markers, --slow

## Labelling tests so you can choose among them {#overview}
As your suite grows, you will want to run groups of tests rather than all of them: the fast ones before every commit, the slow ones overnight, the ones that need a database only when one is available. A marker is a label you attach to a test so that you can pick it out later. pytest has some built-in markers, and you can invent your own. There is one trap, a typing mistake, and an option that catches it.

## marker {#marker}
You attach a marker with `@pytest.mark.something`, such as `@pytest.mark.skip` or a name you invent. A marker does not change what the test does. It gives pytest, and you, a way to find tests and treat them as a group.

Typical custom markers are `slow` for tests that take a long time, `integration` for tests that need other services, and `smoke` for a small set of quick checks. You can put several markers on one test. To mark a whole class or a whole file, write `pytestmark = pytest.mark.slow` at the top.

To select tests, use `-m` with an expression. `pytest -m slow` runs only the slow tests, `pytest -m "not slow"` runs everything else, and `pytest -m "integration and not slow"` combines the two.

## markers {#markers}
In the plural, "markers" also means the list of registered markers in your configuration. To register a marker, you declare its name and a short description in the configuration file, so that pytest knows the name is intentional. Running `pytest --markers` prints every marker pytest knows, with its description, which makes a handy reference for the team.

## strict-markers {#strict-markers}
Here is the trap. Suppose you mistype `@pytest.mark.slwo`. By default pytest takes it as a brand-new marker, gives a quiet warning at most, and moves on. The test is then in the wrong group: `-m slow` does not run it, and `-m "not slow"` does not exclude it. `--strict-markers` turns any unregistered marker name into a hard error, and the whole run stops with a message naming the unknown marker.

Set it permanently in the configuration file, so that nobody has to remember it.

## --slow {#slow}
`--slow` is not a built-in pytest flag. If you type `pytest --slow`, pytest reports an unrecognised argument. To choose slow tests, mark them and use `-m slow`. Some projects add their own `--slow` option with a hook in `conftest.py`. That is a possible design, but it is custom code and not part of pytest.

## A worked example {#example}
Let's mark one test and look at the mark.

```text
import pytest


@pytest.mark.slow
def test_big_export():
    pass


print([m.name for m in test_big_export.pytestmark])
# ['slow']
```

The decorator attached a mark called `slow`, and the list of marks on the test function contains it.

Here is the configuration that registers it:

```text
[tool.pytest.ini_options]
markers = [
    "slow: tests that take more than a second",
    "integration: tests that need a running database",
]
addopts = "--strict-markers"
```

What happens in each situation:

| Situation | Result |
|---|---|
| `pytest -m "not slow"` | Runs every test except those marked `slow`, for a quick check before a commit |
| `pytest -m slow` | Runs only the slow ones, for example overnight |
| A test is marked `@pytest.mark.slwo` | With `--strict-markers`, the run stops: "'slwo' not found in `markers` configuration option" |
| `pytest --slow` | Error: unrecognised arguments |

Count the saving. Say 400 tests take 30 seconds in total, and 20 slow tests take another 10 minutes. Then `-m "not slow"` finishes in 30 seconds. A developer who runs the tests 40 times a day saves about 40 × 10 minutes = 400 minutes, which is more than 6 hours, and the slow set runs once at night.

The typing mistake is the dangerous one. If `slwo` were accepted, a slow test would run on every quick check, and all you would notice is a mysterious slowdown. The strict option turns that into an immediate error that names the problem.

## A common mistake {#mistake}
A common slip is to leave markers unregistered, so the report fills with warnings that people learn to ignore, and a real typing mistake goes unnoticed. Register every marker and turn on strict mode. Another slip is to invent a marker and never use it to select anything. A marker is only useful if someone runs `-m`.

## Key points {#key-points}
- A marker labels a test so that you can select it with `-m`.
- Register each custom marker in the configuration, with a description.
- `--strict-markers` makes an unregistered marker an error and catches typing mistakes. Set it in the configuration file.
- `--slow` is not a built-in flag. Use `-m slow`.
