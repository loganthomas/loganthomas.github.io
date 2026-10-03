---
layout: post
title:  "Heads or Tails of a File"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

On the command line, ``head`` and ``tail`` are second nature.
In Python, I always end up reaching for ``f.readlines()``.
That reads the *whole* file into memory just so I can look at a few lines.
It's fine for a small file,
but slow (or worse) for a big log or data dump.
This post shows how to grab the first few lines with ``itertools.islice``,
the last few with ``collections.deque``,
and the last few of a big file by jumping to the end.

For the examples, we'll use a file with 100,000 lines:

```python
from pathlib import Path

lines = "".join(f"line {i}\n" for i in range(1, 100_001))
Path("server.log").write_text(lines)
```

### The First Few Lines
A file object hands you its lines one at a time.
``itertools.islice`` takes the first ``n`` items and stops,
so Python only reads the lines you ask for:

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

Each line keeps its newline at the end.
Use ``print(line, end="")`` or ``line.rstrip("\n")`` if that gets in the way.

**Aside:** if the file is a CSV headed for pandas,
the ``nrows`` argument does the same job:

```python
import pandas as pd

rows = "".join(f"{i},{i * 10}\n" for i in range(1, 100_001))
Path("data.csv").write_text("id,value\n" + rows)
print(pd.read_csv("data.csv", nrows=3))
```

```
   id  value
0   1     10
1   2     20
2   3     30
```

### The Last Few Lines
A ``deque`` with a ``maxlen`` only holds that many items.
When a new one comes in, the oldest one falls off the other end.
Feed it the whole file, and what's left are the last ``n`` lines:

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

Memory stays small, since it only holds ``n`` lines at a time.
But Python still reads every line in the file to get there.

### Jumping to the End
For a big file, it's much faster to skip to the end.
From there, read backwards in blocks
until you've found enough newlines:

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

A few notes on how it works:
- ``f.seek(0, os.SEEK_END)`` jumps to the end and returns the file size in bytes.
- The file is opened in binary mode (``"rb"``).
  Text mode won't let you seek to any byte you like,
  only to the start, the end, or a spot that ``f.tell()`` gave you.
- The loop keeps going until it has *more* than ``n`` newlines,
  since the last line usually ends with one.
  The extra newline also means the first line in ``data``,
  which may be cut off partway, is never returned.
- ``splitlines()`` drops the newlines,
  so these lines don't end in ``\n``.
- ``.decode()`` assumes the file is UTF-8.
  Pass a different encoding if yours isn't.
- It works on files with fewer than ``n`` lines too.
  ``end`` hits ``0`` and the loop stops.

On the 100,000 line file,
the ``deque`` version took about 5.4 ms on my machine
and the seek version took about 46 µs.
Your numbers will vary.
The gap grows as the file grows,
since the seek version only reads the last few blocks.

### Which One to Use
- **First few lines**: ``islice(f, n)``.
- **Last few lines of a small or medium file**: ``deque(f, maxlen=n)``.
- **Last few lines of a big file**: seek from the end.
- **The whole file at once**: that's when ``readlines()`` makes sense.

### Further Reading
- [``itertools.islice``](https://docs.python.org/3/library/itertools.html#itertools.islice){:target="_blank"}
- [``collections.deque``](https://docs.python.org/3/library/collections.html#collections.deque){:target="_blank"}
- [``io.IOBase.seek``](https://docs.python.org/3/library/io.html#io.IOBase.seek){:target="_blank"}
- [``pandas.read_csv``](https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html){:target="_blank"}
