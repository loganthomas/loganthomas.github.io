---
layout: post
title:  "Python Recipe: A Dictionary of Operations with operator, lambda, and partial"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## A Dictionary of Operations with operator, lambda, and partial

### What you will learn
- How to replace an ``if``/``elif`` chain with a dictionary of functions (a dispatch table)
- How to use the ``operator`` module instead of writing small lambdas
- How to use ``functools.partial`` to pre-fill arguments
- How to mix all three in one lookup table

### Overview
Functions are objects in Python, so they can be stored as values in a ``dict``.
That makes it easy to look up *what to do* based on a string, like a symbol, a command name, or a config option.
I use this pattern for small calculators, rule engines, and CLI subcommands.

### Before: an if/elif Chain
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

This works but grows by two lines for every new operation.

### After: a Dictionary of operator Functions
The ``operator`` module has a function for every Python operator
(``operator.add`` is ``a + b``, ``operator.mul`` is ``a * b``, and so on):

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

The ``calculate`` function becomes a lookup:

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

An unknown operator raises a ``KeyError`` with the bad key, which is usually all the error handling needed:

```python
calculate(1, "^", 2)
```
```
KeyError: '^'
```

``operator.add`` and ``lambda a, b: a + b`` do the same thing,
but the ``operator`` version is faster, has a useful ``repr``, and can be pickled.

### Mixing lambda, operator, and partial
Not every operation is a built-in operator.
Use a ``lambda`` for one-off expressions and ``functools.partial`` to pre-fill arguments of an existing function:

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

``partial(operator.mul, 2)`` is a new function that calls ``operator.mul(2, x)``.
``partial(round, ndigits=2)`` calls ``round(x, ndigits=2)``.

``partial`` is especially handy when the only difference between entries is an argument:

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

Unlike a ``lambda``, a ``partial`` shows what it wraps when printed, which helps when debugging:

```python
print(partial(round, ndigits=2))
```
```
functools.partial(<built-in function round>, ndigits=2)
```

### A Simple Rule Engine
Comparison operators work the same way.
Here, filter rules are stored as plain data (field, operator name, value) and the ``dict`` maps the name to a function:

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
field, op, value = ("score", "gt", 80)

print([row["name"] for row in rows if COMPARE[op](row[field], value)])
```
```
['a', 'c']
```

Note the argument order for ``"in"``.
``operator.contains(a, b)`` is ``b in a``, which reads backwards here, so a ``lambda`` is clearer.

### Folding a List with reduce
``operator`` functions pair well with ``functools.reduce`` to fold a list into one value:

```python
from functools import reduce

print(reduce(operator.add, [1, 2, 3, 4]))
print(reduce(operator.mul, [1, 2, 3, 4]))
```
```
10
24
```

### Conclusion
A ``dict`` of functions replaces a long ``if``/``elif`` chain with a single lookup.
Reach for the ``operator`` module first,
``functools.partial`` when an existing function just needs an argument filled in,
and a ``lambda`` for anything else.

### Further Reading
- [``operator`` documentation](https://docs.python.org/3/library/operator.html){:target="_blank"}
- [``functools.partial``](https://docs.python.org/3/library/functools.html#functools.partial){:target="_blank"}
