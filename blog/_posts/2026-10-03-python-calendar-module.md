---
layout: post
title:  "Python Recipe: Using the calendar Module"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## Using the calendar Module

### What you will learn
- How to print a month (or a whole year) from the command line or a script
- How to find the number of days in a month and check for leap years
- How to look up the day of the week for a date
- How to find the nth weekday of a month (like Thanksgiving)

### Overview
I reach for ``datetime`` for almost everything date related.
But every so often I need something ``datetime`` doesn't answer directly,
like "how many days are in February?" or "what date is the fourth Thursday in November?"
The ``calendar`` module in the standard library handles these without any extra installs.

### Printing a Month
```python
import calendar

print(calendar.month(2026, 10))
```
```
    October 2026
Mo Tu We Th Fr Sa Su
          1  2  3  4
 5  6  7  8  9 10 11
12 13 14 15 16 17 18
19 20 21 22 23 24 25
26 27 28 29 30 31
```

``calendar.prmonth()`` prints directly, and ``calendar.calendar(2026)`` returns the full year.
The same thing works from the terminal:

```bash
python -m calendar 2026 10
```

Weeks start on Monday by default.
To start on Sunday, use a ``TextCalendar`` instead of changing the global setting with ``calendar.setfirstweekday()``:

```python
cal = calendar.TextCalendar(firstweekday=calendar.SUNDAY)
cal.prmonth(2026, 11)
```
```
   November 2026
Su Mo Tu We Th Fr Sa
 1  2  3  4  5  6  7
 8  9 10 11 12 13 14
15 16 17 18 19 20 21
22 23 24 25 26 27 28
29 30
```

### Days in a Month
``calendar.monthrange()`` returns a tuple of the weekday the month starts on and the number of days in the month:

```python
print(calendar.monthrange(2026, 2))
print(calendar.monthrange(2028, 2))
```
```
(calendar.SUNDAY, 28)
(calendar.TUESDAY, 29)
```

The second item is the one I usually want.
It's the easiest way to get the last day of a month:

```python
last_day = calendar.monthrange(2026, 2)[1]
print(last_day)
```
```
28
```

### Leap Years
```python
print(calendar.isleap(2026), calendar.isleap(2028))
print(calendar.leapdays(2000, 2030))  # leap years in [2000, 2030)
```
```
False True
8
```

### Day of the Week
``calendar.weekday()`` returns an integer where Monday is ``0`` and Sunday is ``6``.
Use ``calendar.day_name`` to turn it into something readable:

```python
weekday = calendar.weekday(2026, 10, 3)
print(weekday)
print(calendar.day_name[weekday])
```
```
5
Saturday
```

The module also has lists of day and month names (and abbreviations).
Note that ``month_name`` and ``month_abbr`` have an empty string at index ``0``
so January lines up with ``1``:

```python
print(list(calendar.day_name))
print(list(calendar.month_abbr))
```
```
['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
['', 'Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
```

As of Python 3.12, the day and month constants are enums, so ``calendar.FRIDAY`` and ``calendar.Month.OCTOBER``
still behave like ``4`` and ``10`` but print with a name.

### Finding the nth Weekday of a Month
``calendar.monthcalendar()`` returns a list of weeks.
Each week is a list of seven day numbers, with ``0`` for days outside the month:

```python
from pprint import pprint

pprint(calendar.monthcalendar(2026, 11))
```
```
[[0, 0, 0, 0, 0, 0, 1],
 [2, 3, 4, 5, 6, 7, 8],
 [9, 10, 11, 12, 13, 14, 15],
 [16, 17, 18, 19, 20, 21, 22],
 [23, 24, 25, 26, 27, 28, 29],
 [30, 0, 0, 0, 0, 0, 0]]
```

Pull one column out of that grid, drop the zeros, and you have every occurrence of that weekday in the month.
Thanksgiving (in the US) is the fourth Thursday in November:

```python
def nth_weekday(year, month, weekday, n):
    days = [
        day
        for week in calendar.monthcalendar(year, month)
        if (day := week[weekday]) != 0
    ]
    return days[n - 1]


print(nth_weekday(2026, 11, calendar.THURSDAY, 4))
```
```
26
```

Swap ``days[n - 1]`` for ``days[-1]`` to get the last occurrence instead (like Memorial Day, the last Monday in May).

One more for fun, every Friday the 13th in 2026:

```python
import datetime

fridays = [
    datetime.date(2026, month, 13)
    for month in range(1, 13)
    if calendar.weekday(2026, month, 13) == calendar.FRIDAY
]
print(fridays)
```
```
[datetime.date(2026, 2, 13), datetime.date(2026, 3, 13), datetime.date(2026, 11, 13)]
```

### Conclusion
The ``calendar`` module fills in the gaps that ``datetime`` leaves.
``monthrange()`` gives the number of days in a month,
``isleap()`` checks leap years,
``weekday()`` with ``day_name`` gives the day of the week,
and ``monthcalendar()`` makes "nth weekday of the month" problems a one-liner.

### Further Reading
- [``calendar`` documentation](https://docs.python.org/3/library/calendar.html){:target="_blank"}
