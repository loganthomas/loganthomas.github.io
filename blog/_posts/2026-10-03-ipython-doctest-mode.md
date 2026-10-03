---
layout: post
title:  "From In [1] to >>>"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

I like to put small examples in my docstrings.
With [doctest](https://docs.python.org/3/library/doctest.html){:target="_blank"},
those examples also run as tests.
The catch is that I try things out in IPython first,
and IPython's ``In [1]:`` and ``Out[1]:`` prompts are not the ``>>>`` prompts doctest wants.
Fixing the prompts by hand is slow, and it's easy to miss one.
This post shows how ``%doctest_mode`` handles that for you,
how to paste the result into a docstring,
and how to run the tests.

### The Problem with IPython Prompts
Here is a normal IPython session:

```
In [1]: def add(a, b):
   ...:     return a + b
   ...:

In [2]: add(2, 3)
Out[2]: 5
```

That reads well on screen.
But doctest looks for ``>>>`` at the start of each example.
It has no idea what ``In [2]:`` means,
so this text can't go into a docstring as is.

### Turning On %doctest_mode
The ``%doctest_mode`` magic swaps the prompts.
Run it once to turn it on:

```
In [3]: %doctest_mode
Exception reporting mode: Plain
Doctest mode is: ON
>>> add(2, 3)
5
>>> [add(i, 1) for i in range(3)]
[1, 2, 3]
```

The prompts are now ``>>>`` and ``...``.
The ``Out[n]:`` labels are gone too.
What you see is ready to paste into a docstring.

It also switches errors to the plain traceback style:

```
>>> def divide(a, b):
...     return a / b
...
>>> divide(1, 4)
0.25
>>> divide(1, 0)
Traceback (most recent call last):
  Cell In[8], line 1
    divide(1, 0)
  Cell In[6], line 2 in divide
    return a / b
ZeroDivisionError: division by zero
```

The ``Cell In[...]`` lines in the middle won't match a real run.
That's fine, because you don't need them.
Doctest only checks the ``Traceback (most recent call last):`` line and the last line.
You can swap everything in between for ``...``.

Run the magic again to go back to the normal prompts:

```
>>> %doctest_mode
Exception reporting mode: Context
Doctest mode is: OFF
```

### Pasting the Examples into a Docstring
Now copy the session into each function's docstring:

```python
def add(a, b):
    """
    Add two numbers.

    Examples
    --------
    >>> add(2, 3)
    5
    >>> [add(i, 1) for i in range(3)]
    [1, 2, 3]
    """
    return a + b


def divide(a, b):
    """
    Divide a by b.

    Examples
    --------
    >>> divide(1, 4)
    0.25
    >>> divide(1, 0)
    Traceback (most recent call last):
    ...
    ZeroDivisionError: division by zero
    """
    return a / b
```

I saved this as ``mathutils.py``.
Note that the traceback is now just three lines.

### Running the Tests
The ``doctest`` module can run a file from the command line:

```bash
python -m doctest mathutils.py
```

If all the tests pass, it prints nothing at all.
Add ``-v`` to see what ran.
On Python 3.12, the output ends like this:

```bash
python -m doctest -v mathutils.py
```

```
2 items passed all tests:
   2 tests in mathutils.add
   2 tests in mathutils.divide
4 tests in 3 items.
4 passed and 0 failed.
Test passed.
```

If you use ``pytest``, it can find and run doctests next to the rest of your tests:

```bash
pytest --doctest-modules mathutils.py
```

### Going the Other Way
IPython also works in reverse.
It strips ``>>>`` and ``...`` from code you paste in,
even when doctest mode is off.
So you can copy an example from a docstring or the docs
and paste it straight into IPython:

```
In [1]: >>> x = 2

In [2]: >>> x * 21
Out[2]: 42
```

### Further Reading
- [``%doctest_mode`` magic](https://ipython.readthedocs.io/en/stable/interactive/magics.html#magic-doctest_mode){:target="_blank"}
- [``doctest`` documentation](https://docs.python.org/3/library/doctest.html){:target="_blank"}
- [pytest doctest integration](https://docs.pytest.org/en/stable/how-to/doctest.html){:target="_blank"}
