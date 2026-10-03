---
layout: post
title:  "Python Recipe: Reading the Head or Tail of a File"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## Reading the Head or Tail of a File

### What you will learn
- How to read the first ``n`` lines of a file with ``itertools.islice``
- How to read the last ``n`` lines of a file with ``collections.deque``
- How to read the last ``n`` lines of a large file quickly by seeking from the end

### Overview
On the command line, ``head`` and ``tail`` are second nature.
In Python, I always end up reaching for ``f.readlines()`` which reads the *entire* file into memory just to look at a few lines.
That's fine for small files but slow (or impossible) for large logs and data dumps.

For the examples below, assume a file with 100,000 lines:

```python
from pathlib import Path

Path("server.log").write_text("".join(f"line {i}\n" for i in range(1, 100_001)))
```

### Head
A file object is an iterator over its lines.
``itertools.islice`` takes the first ``n`` items from an iterator and stops,
so only the lines needed are read:

```python
from itertools import islice


def head(path, n=10):
    with open(path) as f:
        return list(islice(f, n))


print(head("server.log", n=3))
```
```
['line 1\n', 'line 2\n', 'line 3\n']
```

Each line keeps its trailing newline.
Use ``print(line, end="")`` or ``line.rstrip("\n")`` if that gets in the way.

If the file is a CSV going into pandas, ``nrows`` does the same thing:

```python
import pandas as pd

pd.read_csv("data.csv", nrows=3)
```

### Tail
A ``deque`` with ``maxlen`` only keeps the last ``maxlen`` items added to it.
Feed it the whole file and what's left are the last ``n`` lines:

```python
from collections import deque


def tail(path, n=10):
    with open(path) as f:
        return list(deque(f, maxlen=n))


print(tail("server.log", n=3))
```
```
['line 99998\n', 'line 99999\n', 'line 100000\n']
```

Memory stays small since only ``n`` lines are held at a time,
but every line in the file is still read.

### Tail for Large Files
For a large file, it's much faster to jump to the end and read backwards in blocks
until enough newlines have been found:

```python
import os


def fast_tail(path, n=10, block_size=4096):
    with open(path, "rb") as f:
        end = f.seek(0, os.SEEK_END)
        data = b""
        while end > 0 and data.count(b"\n") <= n:
            read_size = min(block_size, end)
            end -= read_size
            f.seek(end)
            data = f.read(read_size) + data
    return [line.decode() for line in data.splitlines()[-n:]]


print(fast_tail("server.log", n=3))
```
```
['line 99998', 'line 99999', 'line 100000']
```

A few notes:
- The file is opened in binary mode (``"rb"``) because text mode doesn't support seeking relative to the end.
- The loop reads until it has *more* than ``n`` newlines, since the last line usually ends with one.
- ``splitlines()`` drops the newline characters, so these lines don't have a trailing ``\n``.
- It works on files shorter than ``n`` lines too (``end`` hits ``0`` and the loop stops).

On the 100,000 line file, the ``deque`` version took about 5.5 ms and the seek version about 45 µs on my machine.
The gap grows with the size of the file, since the seek version only reads the last few blocks.

### Conclusion
Avoid ``readlines()`` when you only need a few lines.
Use ``islice(f, n)`` for the head of a file,
``deque(f, maxlen=n)`` for a simple tail,
and seek from the end when the file is large.

### Further Reading
- [``itertools.islice``](https://docs.python.org/3/library/itertools.html#itertools.islice){:target="_blank"}
- [``collections.deque``](https://docs.python.org/3/library/collections.html#collections.deque){:target="_blank"}
