# Configuring pytest in one place

**Covers:** pyproject.toml, ini_options, ini options, addopts, testpaths, setup.cfg, package.json, Dockerfile

## Stop retyping the same flags {#overview}
Left to its defaults, pytest works, but you soon find yourself typing the same options on every run: where the tests are, which markers are allowed, whether to be verbose. Typing them is tiring, and a teammate or the build server may use different ones. A configuration file stores them once, with the project, so that every run behaves the same.

## pyproject.toml {#pyproject-toml}
`pyproject.toml` is the project settings file where modern Python tools, pytest among them, read their options. It is written in TOML, a simple format of sections in square brackets and `key = value` lines. It sits in the root of the project, next to the code. One file can hold the settings for the build tool, the formatter, the type checker and pytest.

## ini_options {#ini-options}
`[tool.pytest.ini_options]` is the table, a named section, in `pyproject.toml` that holds pytest's settings. Everything that you could put on the command line, and some things that you cannot, go under it. The name comes from the older `.ini` format that pytest used before.

## ini options {#ini-options-term}
"ini options" are the pytest settings written in a configuration file, in the `[tool.pytest.ini_options]` section of `pyproject.toml`, or in a plain `pytest.ini`, `tox.ini` or `setup.cfg` file. The word describes the settings themselves, as against command-line flags. Pytest looks for a file in the current folder, and then in the parent folders, and the first one that has pytest settings wins, so check which file is being used with the header line that pytest prints, `configfile: pyproject.toml`.

## addopts {#addopts}
`addopts` is an ini option that lists command-line flags that pytest should apply on every run, without you typing them. For example, `addopts = "-v --strict-markers"`. It is the place for options that you always want, such as strict marker checking or the coverage settings. You can still add more on the command line.

## testpaths {#testpaths}
`testpaths` is an ini option that tells pytest which directories to search for tests by default, so that running plain `pytest` from anywhere in the project looks only in the right place. For example, `testpaths = ["tests"]`. It saves time, since pytest does not walk through large folders such as `node_modules` or `build`, and avoids collecting stray files by mistake.

## setup.cfg {#setup-cfg}
`setup.cfg` is an older settings file for Python projects. Pytest can read it, but the section is named `[tool:pytest]`, with a colon, and not `[tool.pytest.ini_options]`. You will meet it in older projects. If both exist, remember that only one file is used, and a stray section in the wrong one is silently ignored.

## package.json {#package-json}
`package.json` is the description file of a JavaScript project, and pytest does not read settings from it. Projects that mix Python and JavaScript often have both files, and a pytest option placed in `package.json` has no effect at all.

## Dockerfile {#dockerfile}
A Dockerfile holds the instructions for building a Docker image, and pytest ignores any settings placed in it. A Dockerfile may run pytest, with `RUN pytest` or in a build step, but the options belong in `pyproject.toml` so that they are the same inside and outside the container.

## A worked example {#example}
A small configuration in `pyproject.toml`:

```text
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --strict-markers"
markers = [
    "slow: tests that take more than a second",
]
```

What it does:

1. `testpaths` makes plain `pytest` search only the `tests` folder.
2. `addopts` adds `-v` and `--strict-markers` to every run, so each test is listed and unregistered markers are errors.
3. `markers` registers `slow`, so that `--strict-markers` accepts it.

Compare typing and not typing:

| Without the file | With the file |
|---|---|
| `pytest tests -v --strict-markers` | `pytest` |
| 36 characters, easy to forget one flag | 6 characters, always the same flags |
| Teammates and CI may use different options | Everyone and the build server run alike |

Suppose that 10 developers run tests 20 times a day. That is 200 runs, and if the flags were forgotten even once in twenty, 10 runs a day would go without `--strict-markers`, and a typing mistake in a marker could slip by. The file removes that chance completely.

A check on which file pytest uses: the first lines of the output include `rootdir: /path/project` and `configfile: pyproject.toml`. If it says `configfile: setup.cfg`, the settings that you wrote in `pyproject.toml` are not being read.

## A common mistake {#mistake}
Putting pytest options in a file that pytest does not read, such as `package.json` or a Dockerfile, or writing `[tool:pytest]` in `pyproject.toml`. The options are then ignored without any message. Always confirm with the `configfile:` line. Another mistake is keeping two config files with different settings, which makes it unclear which one wins.

## Key points {#key-points}
- Store pytest options in `pyproject.toml` under `[tool.pytest.ini_options]`.
- `addopts` applies flags to every run, and `testpaths` says where to look for tests.
- `setup.cfg` uses `[tool:pytest]`. `package.json` and a Dockerfile are not read.
- Check the `configfile:` line to see which file pytest is using.
