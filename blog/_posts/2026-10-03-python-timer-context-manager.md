---
layout: post
title:  "A Timer That Shows Up on Time"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

Every so often I write a script with a few slow steps,
like loading data, training a model, and saving results.
While it runs, I want one line per step that says what's running and how long it took:

```
Loading data... done in 1.50s
Training model... done in 2.00s
```

The step name should show up as soon as the step starts.
The time should land on the same line when it ends.
That first part is trickier than it looks.
In this post, I build a small timer with ``contextlib``,
show why ``flush=True`` matters,
and reuse the timer as a decorator and as a class.

### The Timer
The ``@contextmanager`` decorator turns a generator into something you can use in a ``with`` block.
The code before ``yield`` runs on the way in.
The code after it runs on the way out:

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

Now wrap each slow step in a ``with`` block:

```python
with timer("Loading data"):
    time.sleep(1.5)

with timer("Training model"):
    time.sleep(2)
```

```
Loading data... done in 1.50s
Training model... done in 2.00s
```

The times come from a real run and will vary a little on your machine.
A few things are going on here:

- ``end=" "`` keeps the cursor on the same line,
  so the time gets added to the end of it.
- ``time.perf_counter()`` is the clock to use for timing code.
  It only moves forward.
  ``time.time()`` follows the system clock,
  which can jump if the clock gets changed.
- ``try``/``finally`` makes sure the time still prints if the block fails.

### Why flush=True?
Python doesn't always write ``print()`` output right away.
It holds the text in a buffer and writes it out later.

When you run a script in a terminal,
the buffer gets written each time a newline is printed.
The first ``print()`` ends with a space, not a newline.
Without ``flush=True``, ``Loading data...`` would sit in the buffer.
It would only show up when the step ends,
at the same time as ``done in 1.50s``.
That defeats the whole point of the message.

When output goes to a pipe or a file,
like ``python train.py | tee log.txt``,
it's worse.
Python waits until the buffer fills up,
which can take a lot of lines.
``flush=True`` sends the text out right away in both cases.

**Aside:** if you don't want to add ``flush=True`` to every ``print()``,
run Python with ``python -u``
or set the ``PYTHONUNBUFFERED=1`` environment variable.
Both turn off this buffering for the whole script.

### Errors Still Get Timed
Thanks to the ``finally``,
a step that fails still reports how long it ran before the error:

```python
with timer("Failing step"):
    time.sleep(0.25)
    raise ValueError("bad data")
```

```
Failing step... done in 0.25s
ValueError: bad data
```

The timer doesn't swallow the error.
It prints the time, and then the error carries on as usual.

### Using It as a Decorator
A context manager built with ``@contextmanager`` also works as a decorator.
That makes it easy to time every call to a function:

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

Each call gets a fresh timer,
so the times don't add up across calls.

### A Class That Keeps the Time
The generator version only prints the time.
Sometimes I want to keep it,
say to log it or put it in a results table.
A class with ``__enter__`` and ``__exit__`` methods can store it on ``self``:

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

print(round(t.elapsed, 2))
```

```
Scoring... done in 0.75s
0.75
```

``__exit__`` runs even if the block fails, just like the ``finally`` above.
It returns ``None``, which is falsy,
so Python doesn't hide the error.
It gets raised as usual after the time prints.

### Further Reading
- [``contextlib.contextmanager``](https://docs.python.org/3/library/contextlib.html#contextlib.contextmanager){:target="_blank"}
- [``print()`` and the ``flush`` argument](https://docs.python.org/3/library/functions.html#print){:target="_blank"}
- [``time.perf_counter()``](https://docs.python.org/3/library/time.html#time.perf_counter){:target="_blank"}
- [The ``-u`` command line option](https://docs.python.org/3/using/cmdline.html#cmdoption-u){:target="_blank"}
