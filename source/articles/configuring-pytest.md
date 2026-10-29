# Configuring pytest in one place

**Covers:** pyproject.toml, ini_options, ini options, addopts, testpaths, setup.cfg, package.json, Dockerfile

## Stop retyping the same flags {#overview}
Left to its defaults, pytest works. Soon, though, you notice that you type the same options on every run: where the tests are, which markers are allowed, whether to be verbose. That is tiring, and a teammate or the build server may type different ones. A configuration file stores the options once, with the project, so every run behaves the same way.

## pyproject.toml {#pyproject-toml}
`pyproject.toml` is the project settings file where modern Python tools, pytest among them, read their options. It is written in TOML, a plain format made of sections in square brackets and `key = value` lines. It sits in the root of the project, next to the code, and one file can hold the settings for the build tool, the formatter, the type checker and pytest.

## ini_options {#ini-options}
`[tool.pytest.ini_options]` is the named section in `pyproject.toml` where pytest's settings go. Anything you could put on the command line goes there, and a few things that you cannot. The name comes from the older `.ini` format that pytest used before.

## ini options {#ini-options-term}
"Ini options" is the name for pytest settings written in a configuration file, as opposed to flags typed on the command line. They can live in the `[tool.pytest.ini_options]` section of `pyproject.toml`, or in a plain `pytest.ini`, `tox.ini` or `setup.cfg` file. Pytest looks in the current folder and then in the parent folders, and the first file with pytest settings wins. To see which one it picked, read the header line pytest prints, `configfile: pyproject.toml`.

## addopts {#addopts}
`addopts` is an ini option holding command-line flags that pytest applies on every run, so you no longer type them. For example, `addopts = "-v --strict-markers"`. Use it for options you always want, such as strict marker checking or coverage settings. You can still add more flags on the command line.

## testpaths {#testpaths}
`testpaths` is an ini option that tells pytest which directories to search for tests by default, so that plain `pytest` run from anywhere in the project looks only in the right place. For example, `testpaths = ["tests"]`. It saves time, because pytest does not walk through large folders such as `node_modules` or `build`, and it avoids collecting stray files by mistake.

## setup.cfg {#setup-cfg}
`setup.cfg` is an older settings file for Python projects. Pytest can read it, but the section is named `[tool:pytest]`, with a colon, and not `[tool.pytest.ini_options]`. You will meet it in older projects. Only one file is used, so remember that a pytest section in the wrong file is silently ignored.

## package.json {#package-json}
`package.json` is the description file of a JavaScript project, and pytest does not read settings from it. Projects that mix Python and JavaScript often have both files, and a pytest option placed in `package.json` has no effect at all.

## Dockerfile {#dockerfile}
A Dockerfile holds the instructions for building a Docker image, and pytest ignores any settings placed in it. A Dockerfile may run pytest, with `RUN pytest` or in a build step, but the options belong in `pyproject.toml` so that they are the same inside and outside the container.

## A worked example {#example}
Here is a small configuration in `pyproject.toml`:

```text
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --strict-markers"
markers = [
    "slow: tests that take more than a second",
]
```

Let's see what each line does.

1. `testpaths` makes plain `pytest` search only the `tests` folder.
2. `addopts` adds `-v` and `--strict-markers` to every run, so each test is listed and unregistered markers are errors.
3. `markers` registers `slow`, so that `--strict-markers` accepts it.

Now compare typing with and without the file:

| Without the file | With the file |
|---|---|
| `pytest tests -v --strict-markers` | `pytest` |
| 32 characters, easy to forget one flag | 6 characters, always the same flags |
| Teammates and CI may use different options | Everyone and the build server run alike |

Suppose 10 developers run tests 20 times a day. That is 200 runs. If the flags were forgotten once in twenty runs, 10 runs a day would go without `--strict-markers`, and a typing mistake in a marker could slip by. The file removes that chance.

You can check which file pytest uses. The first lines of the output include `rootdir: /path/project` and `configfile: pyproject.toml`. If it says `configfile: setup.cfg`, the settings you wrote in `pyproject.toml` are not being read.

## A common mistake {#mistake}
A common slip is to put pytest options in a file pytest does not read, such as `package.json` or a Dockerfile, or to write `[tool:pytest]` in `pyproject.toml`. The options are then ignored without any message. Always confirm with the `configfile:` line. Another slip is keeping two config files with different settings, which makes it unclear which one wins.

## Key points {#key-points}
- Store pytest options in `pyproject.toml` under `[tool.pytest.ini_options]`.
- `addopts` applies flags to every run, and `testpaths` says where to look for tests.
- `setup.cfg` uses `[tool:pytest]`. `package.json` and a Dockerfile are not read.
- Check the `configfile:` line to see which file pytest is using.
