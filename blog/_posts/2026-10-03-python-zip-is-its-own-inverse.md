---
layout: post
title:  "Python Recipe: zip Is Its Own Inverse"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## zip Is Its Own Inverse

### What you will learn
- How to "unzip" a list of pairs with ``zip(*pairs)``
- How to transpose a list of lists with the same trick
- What to watch out for (tuples, uneven lengths, and empty inputs)

### Overview
``zip()`` pairs up items from multiple iterables.
There is no ``unzip()`` function in Python because ``zip()`` already does the job:
call ``zip()`` on the *unpacked* output of ``zip()`` and you get the original sequences back.

### Zipping and Unzipping
```python
names = ["alice", "bob", "carol"]
scores = [90, 72, 85]

pairs = list(zip(names, scores))
print(pairs)
```
```
[('alice', 90), ('bob', 72), ('carol', 85)]
```

Now unzip by passing each pair as a separate argument with ``*``:

```python
new_names, new_scores = zip(*pairs)
print(new_names)
print(new_scores)
```
```
('alice', 'bob', 'carol')
(90, 72, 85)
```

Why does this work?
``zip(*pairs)`` is the same as writing:

```python
zip(("alice", 90), ("bob", 72), ("carol", 85))
```

``zip()`` takes the first item from each tuple (all the names), then the second item from each tuple (all the scores).
So ``zip(*zip(a, b))`` gives back ``a`` and ``b``.

### Transposing a Matrix
The same trick transposes a list of lists, turning rows into columns:

```python
matrix = [
    [1, 2, 3],
    [4, 5, 6],
]
print(list(zip(*matrix)))
```
```
[(1, 4), (2, 5), (3, 6)]
```

### Splitting a Dictionary into Keys and Values
``dict.items()`` yields pairs, so it unzips too:

```python
d = {"a": 1, "b": 2, "c": 3}
keys, values = zip(*d.items())
print(keys, values)
print(dict(zip(keys, values)))
```
```
('a', 'b', 'c') (1, 2, 3)
{'a': 1, 'b': 2, 'c': 3}
```

### Gotchas
**You get tuples back, not lists.**
If you need lists, convert them:

```python
print([list(row) for row in zip(*matrix)])
```
```
[[1, 4], [2, 5], [3, 6]]
```

**Uneven lengths are silently truncated.**
``zip()`` stops at the shortest input, which means the round trip loses data:

```python
print(list(zip(["a", "b", "c"], [1, 2])))
```
```
[('a', 1), ('b', 2)]
```

Use ``strict=True`` (Python 3.10+) to raise an error instead,
or ``itertools.zip_longest`` to pad the shorter input:

```python
from itertools import zip_longest

list(zip(["a", "b", "c"], [1, 2], strict=True))
```
```
ValueError: zip() argument 2 is shorter than argument 1
```
```python
print(list(zip_longest(["a", "b", "c"], [1, 2])))
```
```
[('a', 1), ('b', 2), ('c', None)]
```

**Unzipping an empty list can't be unpacked.**
With no pairs, ``zip(*[])`` is ``zip()``, which yields nothing.
Unpacking it into two names fails:

```python
pairs = []
names, scores = zip(*pairs)
```
```
ValueError: not enough values to unpack (expected 2, got 0)
```

Guard against this if the input can be empty.

### Conclusion
``zip(*pairs)`` reverses ``zip()``.
Use it to unzip a list of pairs, transpose a matrix, or split ``dict.items()`` into keys and values.
Just remember that it returns tuples, truncates to the shortest input (unless ``strict=True``),
and can't be unpacked when the input is empty.

### Further Reading
- [``zip()`` documentation](https://docs.python.org/3/library/functions.html#zip){:target="_blank"}
