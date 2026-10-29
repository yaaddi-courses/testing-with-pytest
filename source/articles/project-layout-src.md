# Project layout: the src layout and editable installs

**Covers:** src layout, src-layout, editable install, PyPI

## Where the tests and the code should live {#overview}
Which folders you create decides which copy of your code the tests import. That sounds minor, but with a wrong layout your tests can pass against code that is not what you ship. A layout with a separate source folder prevents that, and one command makes your package importable while you work. Let's see why this matters, starting with where users get your package.

## PyPI {#pypi}
PyPI, the Python Package Index, is the public website where Python packages are published and from which `pip install` fetches them. When you publish your own library, you upload a built package there. This matters for layout because a published package contains only what the build includes. A test that passes against your working folder may fail against what users actually install.

## src layout {#src-layout}
In a src layout, the installable package lives inside a `src/` folder, as `src/mypackage/`, and the tests sit in a separate `tests/` directory next to it. The project's root folder holds no importable package of its own.

```text
myproject/
  pyproject.toml
  src/
    mypackage/
      __init__.py
      core.py
  tests/
    test_core.py
```

The alternative, the flat layout, puts `mypackage/` directly in the root, beside `tests/`.

## src-layout {#src-layout-hyphen}
You will also see it written "src-layout", with a hyphen, in documents and tools. It means the same thing: the package code is kept in a `src` folder, so that tests import the installed package. Packaging guides use both spellings.

## Why it matters {#why}
With the flat layout, Python puts the current folder on the import path. If you run `pytest` from the root, `import mypackage` finds the folder in front of you, not the installed copy. Two things can then go wrong:
- A file missing from the build, such as a data file or a sub-package left out of the configuration, is still found in the working folder. The tests pass, and the published package is broken.
- A stale installed copy is ignored, and you do not notice.

With the src layout, the package is not in the current folder, so it cannot be imported until it is installed. The tests then always use the installed version, which is the same thing your users will get.

## editable install {#editable-install}
An editable install, made with `pip install -e .`, links your package's source files into your environment instead of copying them. The package behaves as if installed, so imports work from anywhere, but edits to the source take effect at once, with no reinstall. It is the normal way to develop with the src layout.

You run it once in the project root, after creating a virtual environment. If you change the dependencies or the package's metadata in `pyproject.toml`, run it again.

## A worked example {#example}
Let's set up a project and run its tests.

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e ".[test]"
pytest
```

Here is what happens.

1. A virtual environment keeps this project's packages separate.
2. `pip install -e ".[test]"` installs `mypackage` in editable mode, together with the extra test dependencies listed in `pyproject.toml`.
3. `pytest` runs the tests. `import mypackage` now resolves to `src/mypackage`, through the installed link.

Now compare the two layouts with a concrete failure. You add a new file `mypackage/data/rates.json` and forget to list it in the package data. With the flat layout, the tests open the file from the working folder and pass. Users install from PyPI and get a `FileNotFoundError`. With the src layout, the tests import the installed package, which lacks the file, so the test fails at once and you fix the configuration before the release.

Think about the scale. If one release in twenty has such a packaging mistake, a team shipping 10 releases a year would have one broken release every two years, with real users affected. The src layout moves that failure from a user's machine to your test run.

You can check the import with one line: `python -c "import mypackage; print(mypackage.__file__)"`. With an editable install, the path ends in `src/mypackage/__init__.py`.

## A common mistake {#mistake}
A common slip is to use the src layout and then forget the install step, so every test fails with `ModuleNotFoundError`. That error is the layout doing its job. Install first with `pip install -e .`. Another slip is adding `src` to the path by hand instead, which brings back the original problem.

## Key points {#key-points}
- In a src layout, the package is in `src/`, and the tests are in a separate `tests/` folder.
- The layout stops tests from importing the working folder, so they test what you will really ship.
- `pip install -e .` makes an editable install, so edits apply with no reinstall.
- PyPI is where published packages live. Test against the installed package.
