---
layout: post
title:  "Python Recipe: A Timer Context Manager with Print Flushing"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## A Timer Context Manager with Print Flushing

### What you will learn
- How to write a timer context manager with ``contextlib.contextmanager``
- Why ``flush=True`` matters when printing without a newline
- How to reuse the same timer as a decorator
- How to write a class version that keeps the elapsed time

### Overview
When a script has a few slow steps (loading data, training a model, writing results),
I like to see one line per step that says what's running and how long it took:

```
Loading data... done in 1.50s
Training model... done in 2.01s
```

The goal is for ``Loading data...`` to show up *as soon as the step starts*,
and ``done in 1.50s`` to be appended to the same line when it finishes.
That second part is where ``flush=True`` comes in.

### The Timer
```python
import time
from contextlib import contextmanager


@contextmanager
def timer(msg):
    print(f"{msg}...", end=" ", flush=True)
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"done in {time.perf_counter() - start:.2f}s", flush=True)
```

```python
with timer("Loading data"):
    time.sleep(1.5)

with timer("Training model"):
    time.sleep(2)
```
```
Loading data... done in 1.50s
Training model... done in 2.01s
```

A few notes:
- ``end=" "`` keeps the cursor on the same line so the result is appended after the step finishes.
- ``time.perf_counter()`` is the right clock for measuring durations. ``time.time()`` can jump if the system clock changes.
- ``try``/``finally`` makes sure the timing line still prints if the block raises.

### Why flush=True?
``stdout`` is line buffered when writing to a terminal.
Text sits in a buffer until a newline is written (or the buffer fills up).
Because the first ``print()`` uses ``end=" "`` instead of a newline,
``Loading data...`` would stay in the buffer and only appear *after* the step finishes,
all at once with ``done in 1.50s``.
That defeats the purpose of the message.

When ``stdout`` is redirected to a file or piped (e.g. ``python train.py | tee log.txt``, or a job scheduler),
it is block buffered and the delay is even worse.
``flush=True`` forces the text out right away in both cases.

If you'd rather not sprinkle ``flush=True`` everywhere,
running Python with ``python -u`` (or setting ``PYTHONUNBUFFERED=1``) turns off buffering for the whole process.

### Errors Still Get Timed
Thanks to the ``finally``, a failing step still reports how long it ran before raising:

```python
with timer("Failing step"):
    time.sleep(0.25)
    raise ValueError("bad data")
```
```
Failing step... done in 0.25s
Traceback (most recent call last):
  ...
ValueError: bad data
```

### Using It as a Decorator
Context managers built with ``@contextmanager`` can also be used as decorators,
so the same ``timer`` works for timing every call to a function:

```python
@timer("Running step")
def step():
    time.sleep(0.5)


step()
step()
```
```
Running step... done in 0.50s
Running step... done in 0.50s
```

### Class Version (Keeping the Elapsed Time)
The generator version only prints.
If the elapsed time is needed later (logging, a results table),
a class with ``__enter__`` and ``__exit__`` can store it:

```python
class Timer:
    def __init__(self, msg):
        self.msg = msg
        self.elapsed = None

    def __enter__(self):
        print(f"{self.msg}...", end=" ", flush=True)
        self.start = time.perf_counter()
        return self

    def __exit__(self, *exc):
        self.elapsed = time.perf_counter() - self.start
        print(f"done in {self.elapsed:.2f}s", flush=True)


with Timer("Scoring") as t:
    time.sleep(0.75)

print(t.elapsed)
```
```
Scoring... done in 0.75s
0.7531311339698732
```

``__exit__`` returns ``None`` (falsy), so any exception raised inside the block still propagates.

### Conclusion
A few lines with ``@contextmanager`` give a reusable timer for any block of code.
Print the step name with ``end=" "`` and ``flush=True`` so it shows up immediately,
use ``time.perf_counter()`` for the measurement,
and put the final print in a ``finally`` so it always runs.

### Further Reading
- [``contextlib.contextmanager``](https://docs.python.org/3/library/contextlib.html#contextlib.contextmanager){:target="_blank"}
- [``print()`` and the ``flush`` argument](https://docs.python.org/3/library/functions.html#print){:target="_blank"}
