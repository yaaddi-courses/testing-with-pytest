# Running tests in parallel with pytest-xdist

**Covers:** pytest-xdist, -n, CPU

## When the suite is the slow part {#overview}
A test suite that takes ten minutes is a suite that people stop running. Much of that time is often the computer waiting, or using only one of its processor cores while the others sit idle. Running tests in parallel puts the idle cores to work. It is one of the cheapest speed-ups available, with one condition: the tests must not depend on each other.

## CPU {#cpu}
The CPU, or central processing unit, is the processor that carries out a program's instructions. Modern CPUs have several cores, and each core can run one stream of work at a time. A plain `pytest` run uses one core, so on a laptop with 8 cores, 7 of them do nothing while your tests run. The number of cores sets the useful limit for parallel testing, because more workers than cores only compete for time.

## pytest-xdist {#pytest-xdist}
pytest-xdist is a plugin you install with `pip install pytest-xdist`. It runs your tests across several worker processes at the same time, instead of one after another. pytest starts the workers, hands out the tests among them and gathers the results into one report. A big suite can finish several times faster.

It has other modes as well, for example `--dist loadfile`, which sends all the tests from one file to the same worker, and `--looponfail`, which re-runs failing tests whenever files change.

## -n {#n}
`-n` is the xdist option that sets the number of worker processes. `pytest -n 4` uses four workers, and `pytest -n auto` starts one worker for each CPU core. `auto` is a sensible default, because it fits the machine without you counting. On a continuous-integration server that shares its cores with other jobs, a fixed number may be better.

## What parallel runs expose {#shared-state}
Here is the part that surprises people. Parallel runs reveal weak test isolation. Tests that were fine in sequence can fail when they run at the same moment, because they share something:
- a file with a fixed name in a common folder,
- a network port,
- a database table, or
- a global variable that is changed.

Two tests that both write `/tmp/output.txt` overwrite each other's data in parallel, and the failure appears only sometimes. The fix is to give each test its own resources: use `tmp_path` for files, choose a free port for each test and use a separate database or schema for each worker. xdist provides a `worker_id` fixture, with values such as `gw0` and `gw1`, to help with that.

Parallelism also has a cost to set up. Starting 8 worker processes takes a second or two, and each one imports your code. For a suite that takes 5 seconds in total, the overhead can make it slower, so use parallel runs where the suite is long.

## A worked example {#example}
Let's estimate the effect of parallelism on a suite.

```python
tests, seconds_each, workers = 600, 0.5, 4
serial = tests * seconds_each
parallel = serial / workers
print(serial, parallel)
# 300.0 75.0
```

Here is how to read it.

1. 600 tests at half a second each take 300 seconds, which is 5 minutes, in one process.
2. With 4 workers sharing the tests evenly, the ideal time is 300 / 4 = 75 seconds.
3. The real time is a little more, because of start-up and uneven loads. Perhaps 85 seconds.

The speed-up is about 300 / 85, which is 3.5 times, and not the full 4. With 8 workers on an 8-core machine, the ideal is 37.5 seconds, but in practice it is around 50, because some tests take much longer than others and the slowest one decides when the run finishes.

Now the shared-state trap, with a test that fails only in parallel:

```text
def test_writes_report():
    open("/tmp/report.txt", "w").write("A")
    assert open("/tmp/report.txt").read() == "A"
```

In a parallel run, a second test that writes "B" to the same file may run between the write and the read, and then the first test fails with `'B' != 'A'`. If you rewrite both tests to use `tmp_path / "report.txt"`, the clash disappears, because each test gets a different folder.

Here are the commands to try:

```bash
pip install pytest-xdist
pytest -n auto
pytest -n 4 --dist loadfile
```

If a failure appears with `-n auto` and disappears without it, take it as a sign of shared state in the tests, and not as bad luck.

## A common mistake {#mistake}
A common slip is to switch on `-n auto`, see odd failures and switch it off again without finding the cause. The failures were real: the tests depend on each other. Another slip is to set more workers than cores, which makes things slower, not faster.

## Key points {#key-points}
- pytest-xdist runs tests in several processes at once. `-n auto` uses one worker per CPU core.
- The gain is often 3 to 4 times on 4 cores, never the full factor, because of overhead and uneven tests.
- Parallel runs expose shared state. Give each test its own files, ports and data.
- A failure that appears only in parallel is a real isolation bug.
