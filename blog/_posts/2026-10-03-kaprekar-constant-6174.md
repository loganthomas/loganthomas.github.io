---
layout: post
title:  "All Roads Lead to 6174"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: math
---

I came across an interesting concept the other day called
[Kaprekar's routine](https://en.wikipedia.org/wiki/Kaprekar%27s_routine){:target="_blank"}.
It goes like this:

Pick any four-digit number, as long as it isn't one digit repeated (like ``1111``).
Then repeat this:

1. Sort the digits from biggest to smallest.
2. Sort the digits from smallest to biggest.
3. Subtract the smaller number from the bigger one.

No matter where you start, you land on **6174** in seven steps or fewer.
For example, pick ``1234``.
Sorted biggest to smallest, it's ``4321``.
Sorted smallest to biggest, it's ``1234``.
Subtract and keep going:

```
Step 1: 4321 - 1234 = 3087
Step 2: 8730 - 0378 = 8352
Step 3: 8532 - 2358 = 6174
```

Once you reach 6174 (Kaprekar's constant), you're stuck.
You always get ``7641`` as the biggest and ``1467`` as the smallest,
which puts you right back where you started: ``7641 - 1467 = 6174``.

The routine is named after the Indian mathematician D. R. Kaprekar.
In this post, I write the routine in Python and use it to verify these claims.

### One Step at a Time
In Python, we'll start with a single step and build the full routine after that.
Each step pads the number to four digits, sorts, and subtracts.
The ``width`` argument comes in handy if we want to test numbers other than four digits,
but we'll stick with four for now:

```python
def kaprekar_step(n, width=4):
    digits = f"{n:0{width}d}"
    big = int("".join(sorted(digits, reverse=True)))
    small = int("".join(sorted(digits)))
    return big - small


print(kaprekar_step(1234))  # Step 1
print(kaprekar_step(3087))  # Step 2
print(kaprekar_step(8352))  # Step 3
print(kaprekar_step(6174))  # Stuck
```

```
3087
8352
6174
6174
```

### The Full Routine
Now we chain the steps together.
``kaprekar_path`` calls ``kaprekar_step`` until it hits 6174
and saves every number along the way.
Keeping the whole path lets us check two things:
that we reach 6174, and that it takes seven steps or fewer.
The path includes the starting number,
so the number of steps is one less than its length.

```python
KAPREKAR = 6174


def kaprekar_path(n):
    path = [n]
    while path[-1] != KAPREKAR:
        path.append(kaprekar_step(path[-1]))
    return path


path = kaprekar_path(1234)
print(f"{path=}, steps: {len(path) - 1}")

path = kaprekar_path(2026)
print(f"{path=}, steps: {len(path) - 1}")
```

```
path=[1234, 3087, 8352, 6174], steps: 3
path=[2026, 5994, 5355, 1998, 8082, 8532, 6174], steps: 6
```

``1234`` takes three steps and ``2026`` takes six.
Both reach 6174 in seven steps or fewer.

### The Leading Zero Gotcha
Some steps give you fewer than four digits.
Starting from ``2111``, the first step is ``2111 - 1112 = 999``.

The routine treats that as ``0999``, so the next step is ``9990 - 0999 = 8991``.
If you drop the zero and treat ``999`` as a three-digit number,
you get ``999 - 999 = 0`` and never reach 6174.
That's why ``kaprekar_step`` pads the number to four digits before sorting:

```python
print(kaprekar_path(2111))
```

```
[2111, 999, 8991, 8082, 8532, 6174]
```

The "not one digit repeated" rule exists for a similar reason.
``1111`` gives ``1111 - 1111 = 0``, and ``0`` stays ``0`` forever:

```python
print(kaprekar_step(1111))
```

```
0
```

So don't call ``kaprekar_path(1111)``.
It never finishes.

**Aside:** in real code, a quick check up front fails fast instead of hanging:

```python
def kaprekar_path(n):
    if len(set(f"{n:04d}")) == 1:
        raise ValueError(f"{n} repeats one digit and never reaches {KAPREKAR}")
    path = [n]
    while path[-1] != KAPREKAR:
        path.append(kaprekar_step(path[-1]))
    return path


kaprekar_path(1111)
```

```
ValueError: 1111 repeats one digit and never reaches 6174
```

### Checking Every Four-Digit Number
There are only 9,000 four-digit numbers, so it's easy to brute force the claim.
First, ``valid`` keeps each number from ``1000`` to ``9999``
that has at least two different digits.
``set(str(n))`` holds the unique digits,
so a number like ``1111`` has a set of size one and gets skipped.
Then ``Counter`` tallies how many steps each valid number takes.
We subtract one from the path length
because the path includes the starting number, and that doesn't count as a step.
The output shows how many valid numbers there are after skipping invalid ones (8,991),
followed by a bar chart of each step count (0 to 7)
and how many valid numbers needed that many steps:

```python
from collections import Counter

valid = [n for n in range(1000, 10_000) if len(set(str(n))) > 1]
steps = Counter(len(kaprekar_path(n)) - 1 for n in valid)

print(f"{len(valid):,} valid numbers")
for k in sorted(steps):
    bar = "█" * (steps[k] // 50)
    print(f"{k} │{bar} {steps[k]:,}")
```

```
8,991 valid numbers
0 │ 1
1 │███████ 356
2 │██████████ 519
3 │██████████████████████████████████████████ 2,124
4 │██████████████████████ 1,124
5 │███████████████████████████ 1,379
6 │██████████████████████████████ 1,508
7 │███████████████████████████████████████ 1,980
```

Exactly 1,379 numbers needed five steps to hit 6174.
Three steps is the most common,
and the count dips at four before it climbs again toward seven.
The single ``0`` step entry is 6174 itself.

All 8,991 numbers reach 6174, and none take more than seven steps.

### What About Other Lengths?
Three digits work too, with a different constant.
Every three-digit number that isn't one digit repeated lands on **495**:

```python
path = [352]
while path[-1] != 495:
    path.append(kaprekar_step(path[-1], width=3))
print(path)
```

```
[352, 297, 693, 594, 495]
```

Past four digits, there's no single constant.
Three and four are the only lengths that have one.

So 6174 really is special.
Start with almost any four-digit number, sort, subtract, and repeat,
and you'll always end up in the same place.

### Further Reading
- [Kaprekar's routine (Wikipedia)](https://en.wikipedia.org/wiki/Kaprekar%27s_routine){:target="_blank"}
- [6174 (Wikipedia)](https://en.wikipedia.org/wiki/6174){:target="_blank"}
