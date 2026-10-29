# Custom markers and strict registration

**Covers:** marker, strict-markers, --slow

## Labelling tests so you can choose among them {#overview}
As a suite grows, you want to run groups of tests: the fast ones before every commit, the slow ones overnight, the ones that need a database only when one is available. A marker is a label that you attach to a test so that you can select it later. pytest has some built-in markers, and you can invent your own. There is one trap, a typing mistake, and an option that catches it.

## marker {#marker}
A marker is a small tag that you attach to a test with `@pytest.mark.something`, such as `@pytest.mark.skip` or a name that you invent. Markers do not change what a test does by themselves. They give pytest, and you, a way to find and treat tests as groups.

Typical custom markers: `slow` for tests that take a long time, `integration` for tests that need other services and `smoke` for a small set of quick checks. You can put several on one test, and you can mark a whole class or a whole file with `pytestmark = pytest.mark.slow` at the top.

To select tests, use `-m` with an expression: `pytest -m slow` runs only the slow tests, `pytest -m "not slow"` runs everything else and `pytest -m "integration and not slow"` combines them.

## markers {#markers}
In the plural, "markers" also means the list of registered markers in your configuration. Registering a marker means declaring its name, and a short description, in the configuration file, so that pytest knows it is intentional. `pytest --markers` prints every marker that pytest knows, together with its description, which makes a useful reference for the team.

## strict-markers {#strict-markers}
`--strict-markers` is a pytest option that turns any unregistered marker name into a hard error, instead of a quiet warning. Without it, a typing mistake such as `@pytest.mark.slwo` is accepted as a brand-new marker that nobody selects, and the test silently ends up in the wrong group: it is not run with `-m slow`, but it is also not excluded from `-m "not slow"` runs. With the option, the whole run stops with an error that names the unknown marker.

It is best set permanently in the configuration file, so that nobody has to remember it.

## --slow {#slow}
`--slow` is not a built-in pytest flag. If you type `pytest --slow`, pytest reports an unrecognised argument. To choose slow tests, mark them and use `-m slow`. Some projects do add their own `--slow` option through a `conftest.py` hook, which is a possible design, but it is custom code and not part of pytest.

## A worked example {#example}
Mark one test and look at the mark:

```text
import pytest


@pytest.mark.slow
def test_big_export():
    pass


print([m.name for m in test_big_export.pytestmark])
# ['slow']
```

The decorator attached a mark called `slow`, and the list of marks on the test function contains it.

The configuration that registers it:

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

Count the saving: if 400 tests take 30 seconds in total and 20 slow tests take another 10 minutes, then `-m "not slow"` runs in 30 seconds. A developer running tests 40 times a day saves about 40 x 10 minutes = 400 minutes, which is more than 6 hours, and the slow set runs once at night.

The typing mistake is the dangerous one: if `slwo` were accepted, a slow test would run on every quick check, and you would only notice a mysterious slowdown. The strict option turns it into an immediate, clear error.

## A common mistake {#mistake}
Not registering markers, so that the report fills with warnings that people ignore, and a real typing mistake goes unnoticed. Register every marker and turn on strict mode. Another mistake is inventing a marker and then never using it to select anything. A marker is only useful if someone runs `-m`.

## Key points {#key-points}
- A marker labels a test so that you can select it with `-m`.
- Register each custom marker in the configuration, with a description.
- `--strict-markers` makes an unregistered marker an error and catches typing mistakes. Set it in the configuration file.
- `--slow` is not a built-in flag. Use `-m slow`.
