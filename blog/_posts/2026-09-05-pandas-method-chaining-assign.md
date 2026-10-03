---
layout: post
title:  "Don't Break the Chain"
date:   2026-09-05 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

A lot of the pandas code I read (and used to write) looks the same.
Copy the DataFrame, add a column, add another column, filter, group, sort.
Each step saves over the same variable.
It works, but it's hard to read from top to bottom.
In a notebook, it's also easy to break by running cells out of order.

Method chaining puts the whole thing in one expression.
In this post, I rewrite some step-by-step code as a chain.
The key piece is ``.assign()`` with a ``lambda``,
and a habit of naming the lambda's argument ``df_``.
I'll also show why skipping that habit can give you a wrong answer with no error.

Here is the data for the examples:

```python
import pandas as pd

df = pd.DataFrame(
    {
        "date": pd.to_datetime(
            [
                "2026-01-05",
                "2026-01-19",
                "2026-02-02",
                "2026-02-16",
                "2026-03-02",
                "2026-03-16",
            ]
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

### Step by Step
Say I want the total revenue for each month,
but only for orders over $20.
Here's the step-by-step way:

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

### One Chain
Here's the same thing as a chain:

```python
result = (
    df.assign(
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

We get the same result with no temporary variable,
and ``df`` is never changed.
The outer parentheses let each method sit on its own line.
You read it top to bottom, one step per line.

### Why lambda df_?
Inside a chain, the DataFrame changes at every step.
You can pass a function to ``.assign()`` or ``.loc[]``.
When you do, pandas calls it with the DataFrame as it is at that point in the chain.
Naming the argument ``df_`` makes this easy to see.
``df`` is the original.
``df_`` is the one in the middle of the chain.

Using ``df`` inside a chain causes two kinds of problems.
The first is loud.
A column you just made doesn't exist on ``df``:

```python
df.assign(
    revenue=df["price"] * df["qty"],
    big_order=df["revenue"] > 20,
)
```

```
KeyError: 'revenue'
```

The second is worse because it's quiet.
After a filter, ``df`` still has all the rows:

```python
print(
    df.loc[lambda df_: df_["product"] == "widget"].assign(
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

``share_wrong`` divides by the total quantity of every product, not just widgets.
There's no error.
The number is just wrong.
Using ``lambda df_:`` avoids both problems.

### Building on Earlier Columns
``.assign()`` adds its columns in the order you write them.
So a later column can use one made earlier in the same call:

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
``.query()`` is another way to filter.
For simple conditions, it reads better than ``.loc[lambda df_: ...]``.
It also sees columns made earlier in the chain:

```python
print(
    df.assign(revenue=lambda df_: df_["price"] * df_["qty"]).query(
        "revenue > 20 and product == 'widget'"
    )
)
```

```
        date product  price  qty  revenue
0 2026-01-05  widget    2.5   12     30.0
5 2026-03-16  widget    2.5   20     50.0
```

### Custom Steps with .pipe
Sometimes a step doesn't fit any built-in method.
In that case, write a function that takes a DataFrame and returns one.
Then call it with ``.pipe()``.
Any extra arguments get passed along to your function:

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

### pd.col in pandas 3.0
pandas 3.0 added ``pd.col()``.
It points to a column of the DataFrame in the chain, with no ``lambda`` needed:

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

### Further Reading
- [``DataFrame.assign``](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.assign.html){:target="_blank"}
- [``DataFrame.pipe``](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.pipe.html){:target="_blank"}
- [``pandas.col``](https://pandas.pydata.org/docs/reference/api/pandas.col.html){:target="_blank"}
