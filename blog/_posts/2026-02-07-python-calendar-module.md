---
layout: post
title:  "Mark Your calendar"
date:   2026-02-07 08:00:00 -0600
author: Logan Thomas
categories: blog
tags: python
---

I reach for ``datetime`` for almost everything date related.
But every so often I need something it doesn't answer directly,
like "how many days are in February?" or "what date is Thanksgiving this year?"
That's where the ``calendar`` module comes in.
It's in the standard library, so there's nothing to install.
This post covers the parts I use most.
You'll see how to print a month, count its days, and check for leap years.
You'll also find the day of the week and the nth weekday of a month.

### Printing a Month
``calendar.prmonth()`` prints a month as a small text calendar:

```python
import calendar

calendar.prmonth(2026, 10)
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

If you want the text instead of printing it, use ``calendar.month()``.
``calendar.calendar(2026)`` does the same for a whole year.
It also works straight from the terminal:

```bash
python -m calendar 2026 10
```

Weeks start on Monday by default.
To start on Sunday, make a ``TextCalendar``:

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

There's also ``calendar.setfirstweekday()``,
but it changes the setting for everything that uses the module.
A ``TextCalendar`` keeps the change to just that one object.

### Days in a Month
``calendar.monthrange()`` returns two things:
the weekday the month starts on, and how many days it has:

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
``calendar.isleap()`` checks one year.
``calendar.leapdays()`` counts the leap years in a range,
not counting the end year:

```python
print(calendar.isleap(2026), calendar.isleap(2028))
print(calendar.leapdays(2000, 2030))
```

```
False True
8
```

### Day of the Week
``calendar.weekday()`` returns a number, where Monday is ``0`` and Sunday is ``6``.
Use ``calendar.day_name`` to turn it into a word:

```python
weekday = calendar.weekday(2026, 10, 3)
print(weekday)
print(calendar.day_name[weekday])
```

```
5
Saturday
```

The module also has lists of day and month names, plus short versions.
``month_name`` and ``month_abbr`` start with an empty string at index ``0``,
so January lines up with ``1``:

```python
print(list(calendar.day_name))
print(list(calendar.month_abbr))
```

```
['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
['', 'Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
```

Since Python 3.12, the day and month constants are enums.
``calendar.FRIDAY`` still acts like ``4``,
but its repr shows the name.
That's why ``monthrange()`` showed ``calendar.SUNDAY`` above.

### Finding the nth Weekday of a Month
``calendar.monthcalendar()`` returns a month as a list of weeks.
Each week is a list of seven days, with ``0`` for days outside the month:

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

Each column is one weekday.
Pull out a column, drop the zeros,
and you have every time that weekday shows up in the month.
Thanksgiving in the US is the fourth Thursday in November:

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

To get the last one instead, swap ``days[n - 1]`` for ``days[-1]``.
That's how you'd find Memorial Day, the last Monday in May.

One more for fun, every Friday the 13th in 2026:

```python
import datetime

for month in range(1, 13):
    if calendar.weekday(2026, month, 13) == calendar.FRIDAY:
        print(datetime.date(2026, month, 13))
```

```
2026-02-13
2026-03-13
2026-11-13
```

### Summary

| Question                         | Answer                                       |
|----------------------------------|----------------------------------------------|
| Print a month                    | ``calendar.prmonth(year, month)``            |
| How many days are in this month? | ``calendar.monthrange(year, month)[1]``      |
| Is it a leap year?               | ``calendar.isleap(year)``                    |
| What day of the week is it?      | ``calendar.day_name[calendar.weekday(...)]`` |
| When is the nth Thursday?        | ``nth_weekday()`` with ``monthcalendar()``   |

### Further Reading
- [``calendar`` documentation](https://docs.python.org/3/library/calendar.html){:target="_blank"}
