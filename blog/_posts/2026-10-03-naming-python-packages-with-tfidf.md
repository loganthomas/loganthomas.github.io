---
layout: post
title:  "Goodbye, utils.py"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python machine-learning
---

Every project I work on ends up with a ``utils.py``.
Sometimes it's a ``helpers.py`` or a ``misc.py``.
These names say nothing about what's inside.
The code itself usually knows better than I do.
The words that show up a lot in one file,
but not in the rest of the codebase,
are a pretty good description of that file.

That's what [TF-IDF](https://en.wikipedia.org/wiki/Tf%E2%80%93idf){:target="_blank"}
(term frequency, inverse document frequency) measures.
A word scores high for a document when it shows up often in that document
and rarely in the others.
If each source file is a document,
the top words become name ideas.
In this post, I split Python code into words,
score them with scikit-learn's ``TfidfVectorizer``,
and test the idea on the standard library before using it on my own code.

### Splitting Code into Words
The default tokenizer in ``TfidfVectorizer`` is built for plain text.
Code needs a little help.
Names like ``parseResponse`` or ``raw_body`` should be split into words.
Python keywords and builtins (``def``, ``return``, ``self``, ``len``) should be dropped,
since every file has them.
I also drop words with two letters or fewer,
since short names like ``i``, ``x``, and ``fp`` don't say much.

```python
import builtins
import keyword
import re

from sklearn.feature_extraction.text import ENGLISH_STOP_WORDS

IDENTIFIER = re.compile(r"[A-Za-z]+")
CAMEL = re.compile(r"(?<=[a-z])(?=[A-Z])")
STOP_WORDS = (
    ENGLISH_STOP_WORDS
    | set(keyword.kwlist)
    | set(dir(builtins))
    | {"self", "cls", "args", "kwargs"}
)


def tokenize(source):
    words = []
    for ident in IDENTIFIER.findall(source):
        for part in CAMEL.split(ident):
            word = part.lower()
            if len(word) > 2 and word not in STOP_WORDS:
                words.append(word)
    return words


print(tokenize("def parseResponse(self, raw_body): return None"))
print(tokenize("class TarInfo:  # holds the tar header"))
```

```
['parse', 'response', 'raw', 'body']
['tar', 'info', 'holds', 'tar', 'header']
```

``IDENTIFIER`` only matches letters,
so underscores and digits split ``snake_case`` names for free.
``CAMEL`` splits where a lowercase letter is followed by an uppercase one.
Comments and docstrings are kept on purpose.
They often hold the best description of the code.

**Aside:** this simple split doesn't break up acronyms.
A name like ``parseHTTPResponse`` becomes ``parse`` and ``httpresponse``.
Most Python code uses ``snake_case``, so I didn't bother handling it.

### Testing on the Standard Library
Before I trust this on my own code,
I want to see if it can find names that are already good.
The standard library is a nice test set, since its modules are already well named.
The results below are from Python 3.12.
Other versions will shift a little.

```python
import sysconfig
from pathlib import Path

STDLIB = Path(sysconfig.get_paths()["stdlib"])
FILES = [
    "calendar.py",
    "csv.py",
    "gzip.py",
    "smtplib.py",
    "fractions.py",
    "heapq.py",
    "tarfile.py",
    "statistics.py",
    "textwrap.py",
    "uuid.py",
]
docs = {name: (STDLIB / name).read_text() for name in FILES}
```

To make it a real test, I'll add a mystery file.
Here's a ``utils.py`` that I did not name well:

```python
docs["utils.py"] = '''
import random
import time


def _sleep_with_backoff(attempt, base_delay=0.5, max_delay=30.0):
    delay = min(max_delay, base_delay * 2**attempt)
    jitter = random.uniform(0, delay)
    time.sleep(jitter)


def retry(func, attempts=5, exceptions=(ConnectionError,)):
    """Call func, retrying with exponential backoff on failure."""
    for attempt in range(attempts):
        try:
            return func()
        except exceptions:
            _sleep_with_backoff(attempt)
    raise RuntimeError(f"Gave up after {attempts} attempts")
'''
```

### Scoring with TF-IDF
Now pass ``tokenize`` to ``TfidfVectorizer``.
A few settings matter here:

- ``lowercase=False`` is needed.
  By default the text is lowercased before the tokenizer sees it,
  and then ``CAMEL`` has nothing to split.
- ``token_pattern=None`` turns off a warning that the default pattern won't be used.
- ``sublinear_tf=True`` counts ``1 + log(tf)`` instead of the raw count.
  That way a word that shows up 200 times doesn't drown out one that shows up 20 times.

I wrap it in a function since I'll use it again for packages.
It returns one row per document and one column per word:

```python
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer


def tfidf_scores(docs):
    vectorizer = TfidfVectorizer(
        tokenizer=tokenize,
        lowercase=False,
        token_pattern=None,
        sublinear_tf=True,
    )
    X = vectorizer.fit_transform(docs.values())
    return pd.DataFrame(
        X.toarray(),
        index=list(docs),
        columns=vectorizer.get_feature_names_out(),
    )


scores = tfidf_scores(docs)
print(scores.shape)
for name, row in scores.iterrows():
    print(f"{name:<15} {', '.join(row.nlargest(5).index)}")
```

```
(11, 2879)
calendar.py     month, day, calendar, year, locale
csv.py          dialect, fieldnames, delimiter, skipinitialspace, delims
gzip.py         gzip, fileobj, filename, mtime, buffer
smtplib.py      smtp, server, port, resp, ehlo
fractions.py    denominator, numerator, fraction, coprime, exponent
heapq.py        heap, elem, heapreplace, heapify, item
tarfile.py      tarinfo, tar, pax, tarfile, sparse
statistics.py   mean, sigma, statistics, median, dist
textwrap.py     indent, whitespace, chunks, margin, tabs
uuid.py         uuid, mac, getnode, namespace, clock
utils.py        delay, backoff, sleep, attempts, attempt
```

The standard library modules mostly name themselves.
``calendar``, ``gzip``, ``smtp``, ``tar``, ``heap``, and ``uuid``
all show up in their own top five.

Here are the scores for the mystery file:

```python
print(scores.loc["utils.py"].nlargest(8).round(3))
```

```
delay         0.389
backoff       0.342
sleep         0.342
attempts      0.332
attempt       0.292
jitter        0.276
func          0.257
exceptions    0.167
Name: utils.py, dtype: float64
```

``backoff.py`` (or ``retry.py``) is a much better name than ``utils.py``.
I treat the output as a list of ideas.
It still takes a person to pick the name.
But it's a lot easier to pick from a list than from a blank page.

### Naming a Package
The same idea works one level up.
Join every ``.py`` file in a folder into one document,
and each document now stands for a package:

```python
PACKAGES = [
    "json",
    "email",
    "http",
    "logging",
    "sqlite3",
    "unittest",
    "asyncio",
    "importlib",
]
pkg_docs = {
    pkg: "\n".join(path.read_text() for path in sorted((STDLIB / pkg).rglob("*.py")))
    for pkg in PACKAGES
}

pkg_scores = tfidf_scores(pkg_docs)
for name, row in pkg_scores.iterrows():
    print(f"{name:<10} {', '.join(row.nlargest(5).index)}")
```

```
json       nextchar, indent, iterencode, json, infinity
email      defect, defects, cfws, charset, cte
http       cookie, cookies, expires, response, netscape
logging    loggers, config, rollover, filters, critical
sqlite3    sqlite, sql, row, schema, ticks
unittest   mock, suite, tear, autospec, magics
asyncio    fut, waiter, cancelled, cancel, coro
importlib  metadata, loader, traversable, fullname, bootstrap
```

Packages are noisier than single files, since they cover more ground.
Still, ``json``, ``sqlite``, ``mock``, ``suite``, and ``loader`` point the right way.

### Running It on Your Own Code
To use this on a real project, swap the standard library for your code.
Each file in the project is a document,
so the "rare across documents" part is measured against *your* codebase:

```python
PROJECT = Path("src/my_project")
docs = {
    str(path.relative_to(PROJECT)): path.read_text() for path in PROJECT.rglob("*.py")
}
scores = tfidf_scores(docs)
```

Words that show up all over your project,
like your package or company name, get pushed down.
The words that are special to each file float up.

A few tips:
- Add project-wide words (the package name, ``logger``, ``config``) to ``STOP_WORDS``
  if they crowd the results.
- Very small files don't have much to go on.
  Use their results as a starting point.
- If two files share the same top words, they might belong together.

### Further Reading
- [``TfidfVectorizer``](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html){:target="_blank"}
- [TF-IDF (Wikipedia)](https://en.wikipedia.org/wiki/Tf%E2%80%93idf){:target="_blank"}
