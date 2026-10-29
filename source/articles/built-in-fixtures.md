# Built-in fixtures: tmp_path, capsys and monkeypatch

**Covers:** tmp_path, capsys, readouterr, print(), monkeypatch, GUI

## Fixtures that come with the tool {#overview}
pytest ships with a set of ready-made fixtures for the most common testing chores: working with files, checking what a program prints and changing the environment for a moment. They need no installation and no definition. You name them in a test's parameters and they appear. Three of them cover most daily needs.

## tmp_path {#tmp-path}
`tmp_path` is a built-in fixture that gives each test a fresh, empty temporary directory, as a `pathlib.Path` object, to read and write files in. The directory is unique to the test, so tests cannot trip over each other's files, and pytest keeps the most recent few for you to look at after a failure, deleting older ones.

Use it for any code that reads or writes files: write a sample input, run the code and check the output file. It replaces fragile tricks such as writing into the project folder and deleting afterwards.

## print() {#print}
`print()` is Python's built-in way of writing text to the console, which is called standard output. Programs often report results this way. In tests, the output is captured, and not shown unless the test fails, which keeps the test log clean. A program whose result is only printed is harder to test than one that returns a value, which is a good reason to separate the calculation from the printing.

## capsys {#capsys}
`capsys` is a built-in fixture that captures what a test prints to standard output and standard error, so that you can assert on it. Call `capsys.readouterr()` after the code under test has printed, and check the text.

## readouterr {#readouterr}
`readouterr` is the capsys method that returns everything printed so far, as a pair named `out` and `err`. It also clears the buffer, so a second call returns only what was printed since the first. A typical check is `assert capsys.readouterr().out == "hello\n"`. Note the newline: `print` adds one at the end.

## GUI {#gui}
A GUI is a graphical user interface: the windows, buttons and menus that a person clicks. A program with a GUI usually does not print its results, so `capsys` has nothing to capture. To test such a program, test the logic behind the screen directly, and leave a few separate tests for the screen itself, using a tool built for that purpose.

## monkeypatch {#monkeypatch}
`monkeypatch` is a built-in fixture for safely changing things for the length of one test, such as setting an environment variable, replacing a function or changing the current directory. When the test ends, pytest undoes every change automatically, even if the test failed.

The main methods are `setenv` and `delenv` for environment variables, `setattr` for replacing an attribute or function and `chdir` for the directory. Use it to put your code into a known situation, for example "the API key is missing" or "the clock says midnight", without touching the real settings.

## A worked example {#example}
These programs do by hand what two of the fixtures do.

```python
import io
from contextlib import redirect_stdout


def greet():
    print("hello")


buffer = io.StringIO()
with redirect_stdout(buffer):
    greet()
print(repr(buffer.getvalue()))
# 'hello\n'
```

That is the idea behind `capsys.readouterr().out`: the output goes into a buffer, and the test reads it. The text is `'hello\n'`, with the newline that print adds.

```python
import os
from unittest import mock

with mock.patch.dict(os.environ, {"MODE": "test"}):
    print(os.environ["MODE"])
print(os.environ.get("MODE"))
# test
# None
```

That is the idea behind `monkeypatch.setenv("MODE", "test")`: inside the block, the variable has the value `test`. Afterwards, it is gone, so no other test sees it.

In pytest, the real versions look like this:

```text
def test_greeting(capsys):
    greet()
    assert capsys.readouterr().out == "hello\n"

def test_mode(monkeypatch):
    monkeypatch.setenv("MODE", "test")
    assert get_mode() == "test"

def test_writes_file(tmp_path):
    path = tmp_path / "report.txt"
    path.write_text("done")
    assert path.read_text() == "done"
```

Count the code that you did not write: no buffer set-up, no cleanup of the environment and no temporary folder to create and delete. Each test is 2 or 3 lines, and none of them leaves anything behind.

## A common mistake {#mistake}
Changing `os.environ` directly in a test, and then leaving it changed, so that later tests behave differently. Use `monkeypatch`, which always restores. Another mistake is forgetting the newline in the expected output of a print.

## Key points {#key-points}
- `tmp_path` gives each test its own empty directory for files.
- `capsys` captures what a test prints. `capsys.readouterr()` returns `out` and `err`.
- `monkeypatch` changes environment variables, attributes and directories, and restores them automatically.
- Separate calculation from printing, and keep logic behind a GUI testable on its own.
