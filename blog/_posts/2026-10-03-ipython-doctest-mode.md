---
layout: post
title:  "Python Recipe: Writing Doctests with IPython's %doctest_mode"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## Writing Doctests with IPython's %doctest_mode

### What you will learn
- How to switch IPython's prompts to the standard ``>>>`` style with ``%doctest_mode``
- How to copy that output straight into a docstring
- How to run the doctests with ``doctest`` or ``pytest``

### Overview
[Doctests](https://docs.python.org/3/library/doctest.html){:target="_blank"} are examples in a docstring
that look like an interactive Python session.
They double as documentation and as tests.

I write most of my examples in IPython first,
but IPython's ``In [1]:`` and ``Out[1]:`` prompts don't match the ``>>>`` format that doctest expects.
Rewriting the prompts by hand is tedious and easy to get wrong.
``%doctest_mode`` fixes that.

### Turning on %doctest_mode
Here is a normal IPython session:

```
In [1]: def add(a, b):
   ...:     return a + b
   ...:

In [2]: add(2, 3)
Out[2]: 5
```

Run the ``%doctest_mode`` magic to toggle the prompts:

```
In [3]: %doctest_mode
Exception reporting mode: Plain
Doctest mode is: ON
>>> add(2, 3)
5
>>> [add(i, 1) for i in range(3)]
[1, 2, 3]
```

The prompts are now ``>>>`` and ``...``, the ``Out[n]:`` labels are gone,
and exceptions use the plain traceback format.
What's on screen is ready to paste into a docstring.

Run ``%doctest_mode`` again to switch back:

```
>>> %doctest_mode
Exception reporting mode: Context
Doctest mode is: OFF
```

### Pasting Examples into a Docstring
Copy the session into the function's docstring:

```python
# mathutils.py
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

For exceptions, doctest only checks the ``Traceback (most recent call last):`` header and the final line,
so the middle of the traceback can be replaced with ``...``.

### Running the Doctests
Use the ``doctest`` module from the command line.
It prints nothing when everything passes, so add ``-v`` to see the details:

```bash
python -m doctest -v mathutils.py
```
```
...
2 items passed all tests:
   2 tests in mathutils.add
   2 tests in mathutils.divide
4 tests in 3 items.
4 passed and 0 failed.
Test passed.
```

Or let ``pytest`` collect them along with the rest of the test suite:

```bash
pytest --doctest-modules mathutils.py
```

### Going the Other Way
IPython also strips ``>>>`` and ``...`` prompts from pasted code,
even when doctest mode is off.
That means examples copied from documentation (or a docstring) can be pasted into IPython as is:

```
In [1]: >>> x = 2

In [2]: >>> x * 21
Out[2]: 42
```

### Conclusion
``%doctest_mode`` toggles IPython between its own prompts and the standard ``>>>`` prompts.
Turn it on, work through the example, paste the session into a docstring,
and run it with ``python -m doctest`` or ``pytest --doctest-modules``.

### Further Reading
- [``%doctest_mode`` magic](https://ipython.readthedocs.io/en/stable/interactive/magics.html#magic-doctest_mode){:target="_blank"}
- [``doctest`` documentation](https://docs.python.org/3/library/doctest.html){:target="_blank"}
