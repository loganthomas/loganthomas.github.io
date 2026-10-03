---
layout: post
title:  "Python Recipe: Pandas Method Chaining with .assign and df_"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## Pandas Method Chaining with .assign and df_

### What you will learn
- How to rewrite step-by-step pandas code as a single method chain
- How to use ``.assign()`` with a ``lambda df_:`` to reference the intermediate DataFrame
- Why referencing the original ``df`` inside a chain can silently give wrong answers
- How to use ``.loc``, ``.query()``, and ``.pipe()`` in a chain

### Overview
A lot of pandas code looks like this: copy the DataFrame, add a column, add another column, filter,
group, sort, each step reassigning the same variable.
It works, but it's hard to read top to bottom and easy to break when running notebook cells out of order.

Method chaining puts the whole transformation in one expression.
The key piece is ``.assign()`` with a ``lambda``, and the convention of naming the lambda's argument ``df_``.

Here is the data for the examples:

```python
import pandas as pd

df = pd.DataFrame(
    {
        "date": pd.to_datetime(
            ["2026-01-05", "2026-01-19", "2026-02-02", "2026-02-16", "2026-03-02", "2026-03-16"]
        ),
        "product": ["widget", "gadget", "widget", "gizmo", "gadget", "widget"],
        "price": [2.50, 10.00, 2.50, 7.25, 10.00, 2.50],
        "qty": [12, 3, 4, 6, 1, 20],
    }
)
print(df)
```
```
        date product  price  qty
0 2026-01-05  widget   2.50   12
1 2026-01-19  gadget  10.00    3
2 2026-02-02  widget   2.50    4
3 2026-02-16   gizmo   7.25    6
4 2026-03-02  gadget  10.00    1
5 2026-03-16  widget   2.50   20
```

### Before: Step by Step
Total revenue per month, only counting orders over $20:

```python
tmp = df.copy()
tmp["revenue"] = tmp["price"] * tmp["qty"]
tmp["month"] = tmp["date"].dt.month_name()
tmp = tmp[tmp["revenue"] > 20]
tmp = tmp.groupby("month", as_index=False)["revenue"].sum()
tmp = tmp.sort_values("revenue", ascending=False)
print(tmp)
```
```
      month  revenue
1   January     60.0
2     March     50.0
0  February     43.5
```

### After: One Chain
```python
result = (
    df
    .assign(
        revenue=lambda df_: df_["price"] * df_["qty"],
        month=lambda df_: df_["date"].dt.month_name(),
    )
    .loc[lambda df_: df_["revenue"] > 20]
    .groupby("month", as_index=False)["revenue"]
    .sum()
    .sort_values("revenue", ascending=False)
)
print(result)
```
```
      month  revenue
1   January     60.0
2     March     50.0
0  February     43.5
```

Same result, no temporary variable, and ``df`` is never modified.
Wrapping the chain in parentheses lets each method sit on its own line.

### Why lambda df_?
Inside a chain, the DataFrame changes at every step.
When ``.assign()`` (or ``.loc[]``) is given a function, pandas calls it with the DataFrame *at that point in the chain*.
Naming the argument ``df_`` is a convention that makes this obvious:
``df`` is the original, ``df_`` is the intermediate one.

Referencing ``df`` directly inside a chain causes two kinds of problems.

**A column created earlier in the chain doesn't exist on ``df``:**

```python
(
    df
    .assign(revenue=df["price"] * df["qty"])
    .assign(big_order=df["revenue"] > 20)
)
```
```
KeyError: 'revenue'
```

**Worse, it can silently compute the wrong thing.**
After a filter, ``df`` still has *all* the rows:

```python
print(
    df
    .loc[lambda df_: df_["product"] == "widget"]
    .assign(
        share_wrong=df["qty"] / df["qty"].sum(),
        share_right=lambda df_: df_["qty"] / df_["qty"].sum(),
    )
)
```
```
        date product  price  qty  share_wrong  share_right
0 2026-01-05  widget    2.5   12     0.260870     0.333333
2 2026-02-02  widget    2.5    4     0.086957     0.111111
5 2026-03-16  widget    2.5   20     0.434783     0.555556
```

``share_wrong`` divides by the total quantity of *every* product, not just widgets.
There's no error, the number is just wrong.
Using ``lambda df_:`` avoids both problems.

### Referencing Columns from the Same .assign
Keyword arguments in ``.assign()`` are applied in order,
so a later column can use one created earlier in the same call:

```python
print(
    df.assign(
        revenue=lambda df_: df_["price"] * df_["qty"],
        revenue_share=lambda df_: df_["revenue"] / df_["revenue"].sum(),
    ).round({"revenue_share": 3})
)
```
```
        date product  price  qty  revenue  revenue_share
0 2026-01-05  widget   2.50   12     30.0          0.173
1 2026-01-19  gadget  10.00    3     30.0          0.173
2 2026-02-02  widget   2.50    4     10.0          0.058
3 2026-02-16   gizmo   7.25    6     43.5          0.251
4 2026-03-02  gadget  10.00    1     10.0          0.058
5 2026-03-16  widget   2.50   20     50.0          0.288
```

### Filtering with .query
``.query()`` is an alternative to ``.loc[lambda df_: ...]`` that reads well for simple conditions.
It also sees columns created earlier in the chain:

```python
print(
    df
    .assign(revenue=lambda df_: df_["price"] * df_["qty"])
    .query("revenue > 20 and product == 'widget'")
)
```
```
        date product  price  qty  revenue
0 2026-01-05  widget    2.5   12     30.0
5 2026-03-16  widget    2.5   20     50.0
```

### Custom Steps with .pipe
When a step doesn't fit a built-in method, write a function that takes and returns a DataFrame,
then call it with ``.pipe()``.
Extra arguments are passed through:

```python
def add_revenue(df_, discount=0.0):
    return df_.assign(revenue=df_["price"] * df_["qty"] * (1 - discount))


print(df.pipe(add_revenue, discount=0.1).head(3))
```
```
        date product  price  qty  revenue
0 2026-01-05  widget    2.5   12     27.0
1 2026-01-19  gadget   10.0    3     27.0
2 2026-02-02  widget    2.5    4      9.0
```

### pd.col (pandas 3.0+)
pandas 3.0 added ``pd.col()``, which refers to a column of the intermediate DataFrame without a ``lambda``:

```python
print(df.assign(revenue=pd.col("price") * pd.col("qty")).head(3))
```
```
        date product  price  qty  revenue
0 2026-01-05  widget    2.5   12     30.0
1 2026-01-19  gadget   10.0    3     30.0
2 2026-02-02  widget    2.5    4     10.0
```

On older versions of pandas, stick with ``lambda df_:``.

### Conclusion
Method chaining turns a series of reassignments into one readable expression that doesn't modify the original data.
Inside the chain, always reference the intermediate DataFrame with ``lambda df_:`` (or ``pd.col()``),
never the original ``df``,
or you risk a ``KeyError`` at best and a silently wrong answer at worst.

### Further Reading
- [``DataFrame.assign``](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.assign.html){:target="_blank"}
- [``DataFrame.pipe``](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.pipe.html){:target="_blank"}
