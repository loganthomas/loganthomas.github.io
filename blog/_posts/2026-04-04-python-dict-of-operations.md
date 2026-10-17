---
layout: post
title:  "Swap Your elif Chain for a dict"
date:   2026-04-04 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

Every so often I write an ``if``/``elif`` chain that keeps growing.
Each new case adds two more lines, and the function gets harder to read.
There's a simpler way.
Functions are objects in Python, so you can store them as values in a ``dict``.
Then you look up *what to do* by a string, like a symbol or a command name.
This post shows how to build that lookup table
with the ``operator`` module, ``lambda``, and ``functools.partial``.

### The if/elif Chain
Here's a tiny calculator:

```python
def calculate(a, op, b):
    if op == "+":
        return a + b
    elif op == "-":
        return a - b
    elif op == "*":
        return a * b
    elif op == "/":
        return a / b
    else:
        raise ValueError(f"Unknown operator: {op}")
```

It works.
But it grows by two lines for every new operation.

### A dict of operator Functions
The ``operator`` module has a function for most of Python's operators.
``operator.add(a, b)`` is ``a + b``, ``operator.mul(a, b)`` is ``a * b``, and so on.
Put them in a ``dict`` keyed by symbol:

```python
import operator

OPS = {
    "+": operator.add,
    "-": operator.sub,
    "*": operator.mul,
    "/": operator.truediv,
    "//": operator.floordiv,
    "%": operator.mod,
    "**": operator.pow,
}

print(OPS["+"](2, 3))
print(OPS["**"](2, 10))
```

```
5
1024
```

Now ``calculate`` is a single lookup:

```python
def calculate(a, op, b):
    return OPS[op](a, b)


print(calculate(7, "//", 2))

a, op, b = "12 * 4".split()
print(calculate(int(a), op, int(b)))
```

```
3
48
```

Adding a new operation is one more line in the ``dict``.
An unknown symbol raises a ``KeyError`` that names the bad key.
That's often all the error handling you need:

```python
calculate(1, "^", 2)
```

```
KeyError: '^'
```

**Aside:** you could write ``lambda a, b: a + b`` instead of ``operator.add``.
It does the same math, but the ``operator`` version has a few perks.
It runs a bit faster since it's written in C.
It also has a clear name when printed,
and it can be pickled, which a ``lambda`` can't:

```python
print(operator.add)
```

```
<built-in function add>
```

### Mixing in lambda and partial
Not every action is a built-in operator.
Use a ``lambda`` for a quick one-off expression.
Use ``functools.partial`` when an existing function just needs an argument filled in:

```python
from functools import partial

TRANSFORMS = {
    "square": lambda x: x**2,
    "negate": operator.neg,
    "double": partial(operator.mul, 2),
    "half": partial(operator.mul, 0.5),
    "round2": partial(round, ndigits=2),
}

for name, func in TRANSFORMS.items():
    print(f"{name:>7}: {func(3.14159)}")
```

```
 square: 9.869587728099999
 negate: -3.14159
 double: 6.28318
   half: 1.570795
 round2: 3.14
```

``partial(operator.mul, 2)`` is a new function.
Calling it with ``x`` runs ``operator.mul(2, x)``.
In the same way, ``partial(round, ndigits=2)`` runs ``round(x, ndigits=2)``.

``partial`` shines when the only difference between entries is one argument:

```python
PARSERS = {
    "bin": partial(int, base=2),
    "oct": partial(int, base=8),
    "hex": partial(int, base=16),
}

print(PARSERS["bin"]("1010"), PARSERS["oct"]("17"), PARSERS["hex"]("ff"))
```

```
10 15 255
```

Unlike a ``lambda``, a ``partial`` shows what it wraps when you print it.
That helps when debugging:

```python
print(partial(round, ndigits=2))
```

```
functools.partial(<built-in function round>, ndigits=2)
```

### A Small Rule Engine
Comparisons work the same way.
Here, each filter rule is plain data: a field, an operator name, and a value.
The ``dict`` turns the name into a function:

```python
COMPARE = {
    "eq": operator.eq,
    "gt": operator.gt,
    "lt": operator.lt,
    "in": lambda value, options: value in options,
}

rows = [
    {"name": "a", "score": 90},
    {"name": "b", "score": 72},
    {"name": "c", "score": 85},
]
rules = [("score", "gt", 80), ("name", "in", ["b", "c"])]

for field, op, value in rules:
    print([row["name"] for row in rows if COMPARE[op](row[field], value)])
```

```
['a', 'c']
['b', 'c']
```

Note the ``lambda`` for ``"in"``.
The ``operator`` module does have ``operator.contains(a, b)``,
but it means ``b in a``.
The arguments are flipped from how the rule reads,
so a ``lambda`` is clearer here.

### Folding a List with reduce
``operator`` functions also pair well with ``functools.reduce``.
It folds a list down to one value:

```python
from functools import reduce

print(reduce(operator.add, [1, 2, 3, 4]))
print(reduce(operator.mul, [1, 2, 3, 4]))
```

```
10
24
```

### Further Reading
- [``operator`` documentation](https://docs.python.org/3/library/operator.html){:target="_blank"}
- [``functools.partial``](https://docs.python.org/3/library/functools.html#functools.partial){:target="_blank"}
- [``functools.reduce``](https://docs.python.org/3/library/functools.html#functools.reduce){:target="_blank"}
