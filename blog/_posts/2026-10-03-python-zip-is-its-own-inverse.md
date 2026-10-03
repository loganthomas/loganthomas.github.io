---
layout: post
title:  "zip Undoes Itself"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

Every so often I have a list of pairs and I want two separate lists back.
My first instinct is to look for an ``unzip()`` function.
Python doesn't have one, and it doesn't need one.
``zip()`` can undo itself.
This post shows how that works,
how the same trick flips rows and columns,
and the few spots where it can trip you up.

### Zipping and Unzipping
``zip()`` pairs up items from two or more lists:

```python
names = ["alice", "bob", "carol"]
scores = [90, 72, 85]

pairs = list(zip(names, scores))
print(pairs)
```

```
[('alice', 90), ('bob', 72), ('carol', 85)]
```

To unzip, call ``zip()`` again.
The ``*`` passes each pair as its own argument:

```python
names_again, scores_again = zip(*pairs)
print(names_again)
print(scores_again)
```

```
('alice', 'bob', 'carol')
(90, 72, 85)
```

### Why It Works
The ``*`` spreads the list out.
So ``zip(*pairs)`` is the same as writing this:

```python
zip(("alice", 90), ("bob", 72), ("carol", 85))
```

Now ``zip()`` sees three tuples.
It takes the first item from each one, which gives all the names.
Then it takes the second item from each one, which gives all the scores.
That's why ``zip(*zip(a, b))`` hands you back ``a`` and ``b``.

### Flipping Rows and Columns
The same trick works on a list of lists.
Each inner list is a row,
and ``zip(*rows)`` turns the rows into columns:

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

In math terms, that's a transpose.

### Splitting a dict into Keys and Values
``dict.items()`` gives you pairs, so it unzips too:

```python
ages = {"alice": 31, "bob": 27, "carol": 45}
people, years = zip(*ages.items())
print(people)
print(years)
print(dict(zip(people, years)))
```

```
('alice', 'bob', 'carol')
(31, 27, 45)
{'alice': 31, 'bob': 27, 'carol': 45}
```

**Aside:** if you only need the keys and values on their own,
``list(ages)`` and ``list(ages.values())`` work fine.
The ``zip()`` version is handy when you want both in one line.

### Watch Out for These
**You get tuples back, not lists.**
If you need lists, convert them:

```python
print([list(row) for row in zip(*matrix)])
```

```
[[1, 4], [2, 5], [3, 6]]
```

**Uneven lengths get cut short.**
``zip()`` stops at the shortest input.
It doesn't warn you, so the round trip quietly drops data:

```python
print(list(zip(["a", "b", "c"], [1, 2])))
```

```
[('a', 1), ('b', 2)]
```

Since Python 3.10, you can pass ``strict=True`` to raise an error instead:

```python
list(zip(["a", "b", "c"], [1, 2], strict=True))
```

```
ValueError: zip() argument 2 is shorter than argument 1
```

Or use ``itertools.zip_longest()`` to fill the gaps.
It fills with ``None`` unless you pick a ``fillvalue``:

```python
from itertools import zip_longest

print(list(zip_longest(["a", "b", "c"], [1, 2])))
```

```
[('a', 1), ('b', 2), ('c', None)]
```

**An empty list can't be unpacked.**
With no pairs, ``zip(*[])`` is just ``zip()``, which yields nothing.
So there's nothing to split into two names:

```python
pairs = []
names, scores = zip(*pairs)
```

```
ValueError: not enough values to unpack (expected 2, got 0)
```

If your list might be empty, check for that first.

### Further Reading
- [``zip()`` documentation](https://docs.python.org/3/library/functions.html#zip){:target="_blank"}
- [``itertools.zip_longest()``](https://docs.python.org/3/library/itertools.html#itertools.zip_longest){:target="_blank"}
- [PEP 618: Add Optional Length-Checking To zip](https://peps.python.org/pep-0618/){:target="_blank"}
