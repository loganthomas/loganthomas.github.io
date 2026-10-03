---
layout: post
title:  "What's So Special About 6174?"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## What's So Special About 6174?

### What you will learn
- What Kaprekar's routine is and why every four-digit number ends up at 6174
- How to implement the routine in Python (and the leading zero gotcha)
- How to check every four-digit number to see how many steps each one takes

### Overview
Pick any four-digit number that uses at least two different digits.
Then repeat these steps:
1. Arrange the digits in descending order to make the biggest number possible.
2. Arrange the digits in ascending order to make the smallest number possible.
3. Subtract the smaller number from the bigger one.

No matter where you start, you will reach **6174** in at most seven steps.
Once there, you're stuck: ``7641 - 1467 = 6174``.

This is known as [Kaprekar's routine](https://en.wikipedia.org/wiki/6174){:target="_blank"},
after the Indian mathematician D. R. Kaprekar,
and 6174 is called Kaprekar's constant.

### One Step
```python
def kaprekar_step(n):
    digits = f"{n:04d}"
    big = int("".join(sorted(digits, reverse=True)))
    small = int("".join(sorted(digits)))
    return big - small


print(kaprekar_step(3524))
print(kaprekar_step(6174))
```
```
3087
6174
```

``3524`` becomes ``5432 - 2345 = 3087``.
And ``6174`` maps right back to itself.

### The Full Routine
Keep stepping until 6174 shows up:

```python
KAPREKAR = 6174


def kaprekar_path(n):
    path = [n]
    while path[-1] != KAPREKAR:
        path.append(kaprekar_step(path[-1]))
    return path


print(kaprekar_path(3524))
print(kaprekar_path(2026))
```
```
[3524, 3087, 8352, 6174]
[2026, 5994, 5355, 1998, 8082, 8532, 6174]
```

### The Leading Zero Gotcha
Some steps produce a number with fewer than four digits.
Starting from ``2111``, the first step is ``2111 - 1112 = 999``.

The routine treats that as ``0999``, so the next step is ``9990 - 0999 = 8991``.
If the zero is dropped and ``999`` is treated as a three-digit number,
you get ``999 - 999 = 0`` and never reach 6174.

That's why ``kaprekar_step`` formats the number with ``f"{n:04d}"`` before sorting the digits:

```python
print(kaprekar_path(2111))
```
```
[2111, 999, 8991, 8082, 8532, 6174]
```

The "at least two different digits" rule exists for the same reason.
A repdigit like ``1111`` gives ``1111 - 1111 = 0``, and ``0`` stays ``0`` forever:

```python
print(kaprekar_step(1111))
```
```
0
```

(Calling ``kaprekar_path(1111)`` would loop forever, so don't.)

### Checking Every Four-Digit Number
It's easy to brute force the claim.
Count how many steps every valid four-digit number needs:

```python
from collections import Counter

valid = [n for n in range(1000, 10_000) if len(set(str(n))) > 1]
steps = Counter(len(kaprekar_path(n)) - 1 for n in valid)

print(len(valid))
for k in sorted(steps):
    print(k, steps[k])
```
```
8991
0 1
1 356
2 519
3 2124
4 1124
5 1379
6 1508
7 1980
```

All 8,991 numbers reach 6174, and none take more than seven steps.
The single ``0`` step entry is 6174 itself.

### Does It Work for Three Digits?
It does, with a different constant.
Three-digit numbers all converge to **495**:

```python
def kaprekar_step_3(n):
    digits = f"{n:03d}"
    return int("".join(sorted(digits, reverse=True))) - int("".join(sorted(digits)))


path = [352]
while path[-1] != 495:
    path.append(kaprekar_step_3(path[-1]))
print(path)
```
```
[352, 297, 693, 594, 495]
```

For five or more digits, there's no single constant.
The routine falls into cycles instead.

### Conclusion
6174 is the fixed point of Kaprekar's routine for four-digit numbers:
sort the digits descending, sort them ascending, subtract, repeat.
Every four-digit number with at least two distinct digits gets there in seven steps or fewer.
When implementing it, pad to four digits (``f"{n:04d}"``) so numbers like ``999`` are handled as ``0999``.

### Further Reading
- [6174 (Wikipedia)](https://en.wikipedia.org/wiki/6174){:target="_blank"}
- [Kaprekar's routine (Wikipedia)](https://en.wikipedia.org/wiki/Kaprekar%27s_routine){:target="_blank"}
