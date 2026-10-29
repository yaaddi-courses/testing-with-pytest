# Built-in fixtures: tmp_path, capsys and monkeypatch

**Covers:** tmp_path, capsys, readouterr, print(), monkeypatch, GUI

## Fixtures that come with the tool {#overview}
Almost every project needs the same few things in its tests: a place to write files, a way to check what a program prints, and a way to change the environment for a moment. pytest ships with ready-made fixtures for exactly these chores. You install nothing and define nothing. You name the fixture in a test's parameters, and it appears. Three of them cover most daily needs.

## tmp_path {#tmp-path}
`tmp_path` hands each test a fresh, empty temporary directory, as a `pathlib.Path` object, where it can read and write files. The directory belongs to that test alone, so tests cannot trip over each other's files. pytest keeps the most recent few directories so that you can look inside after a failure, and it deletes older ones.

Reach for it whenever the code reads or writes files: write a sample input, run the code and check the output file. It replaces fragile tricks such as writing into the project folder and deleting afterwards.

## print() {#print}
`print()` is Python's built-in way to write text to the console, known as standard output. Programs often report results this way. During a test, pytest captures that output and shows it only if the test fails, which keeps the log clean. A program whose result is only printed is harder to test than one that returns a value, which is a good reason to keep the calculation and the printing apart.

## capsys {#capsys}
What if the printed text is exactly what you want to check? That is the job of `capsys`. It captures what a test prints to standard output and standard error, so you can assert on it. Call `capsys.readouterr()` after the code under test has printed, and check the text.

## readouterr {#readouterr}
`readouterr` is the capsys method that returns everything printed so far, as a pair named `out` and `err`. It also empties the buffer, so a second call returns only what was printed since the first. A typical check is `assert capsys.readouterr().out == "hello\n"`. Notice the newline: `print` adds one at the end.

## GUI {#gui}
A GUI is a graphical user interface, meaning the windows, buttons and menus that a person clicks. A program with a GUI usually does not print its results, so `capsys` has nothing to capture. The usual approach is to test the logic behind the screen directly, and to keep a few separate tests for the screen itself, using a tool made for that.

## monkeypatch {#monkeypatch}
`monkeypatch` lets you change something for the length of one test, such as an environment variable, a function or the current directory. When the test ends, pytest undoes every change automatically, even if the test failed.

The main methods are `setenv` and `delenv` for environment variables, `setattr` for replacing an attribute or function, and `chdir` for the directory. They let you put your code into a known situation, for example "the API key is missing" or "the clock says midnight", without touching the real settings.

## A worked example {#example}
Before we use the fixtures, let's do by hand what two of them do.

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

That is the idea behind `capsys.readouterr().out`. The output goes into a buffer, and the test reads it. The text is `'hello\n'`, with the newline that print adds.

```python
import os
from unittest import mock

with mock.patch.dict(os.environ, {"MODE": "test"}):
    print(os.environ["MODE"])
print(os.environ.get("MODE"))
# test
# None
```

That is the idea behind `monkeypatch.setenv("MODE", "test")`. Inside the block the variable has the value `test`. Afterwards it is gone, so no other test sees it.

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

Count the code you did not have to write: no buffer set-up, no cleanup of the environment, and no temporary folder to create and delete. Each test is 2 or 3 lines, and none of them leaves anything behind.

## A common mistake {#mistake}
A common slip is to change `os.environ` directly in a test and leave it changed, so that later tests behave differently. Use `monkeypatch`, which always restores. Another slip is forgetting the newline in the expected output of a print.

## Key points {#key-points}
- `tmp_path` gives each test its own empty directory for files.
- `capsys` captures what a test prints, and `capsys.readouterr()` returns `out` and `err`.
- `monkeypatch` changes environment variables, attributes and directories, and restores them automatically.
- Keep calculation apart from printing, and keep the logic behind a GUI testable on its own.
