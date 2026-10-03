---
layout: post
title:  "Naming Things Is Hard: Using TF-IDF to Name Python Modules and Packages"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python machine-learning
---

## Naming Things Is Hard: Using TF-IDF to Name Python Modules and Packages

### What you will learn
- How to tokenize Python source code into words (splitting ``snake_case`` and ``camelCase``)
- How to use scikit-learn's ``TfidfVectorizer`` on source files
- How to read the top TF-IDF terms as name suggestions for a file, module, or package

### Overview
Every project I work on ends up with a ``utils.py`` (or a ``helpers.py``, or a ``misc.py``).
These names say nothing about what's inside.
The code itself usually knows better than I do: the words that show up a lot in one file,
but not in the rest of the codebase, are a pretty good description of that file.

That is exactly what [TF-IDF](https://en.wikipedia.org/wiki/Tf%E2%80%93idf){:target="_blank"}
(term frequency, inverse document frequency) measures.
A word scores high for a document when it appears often in that document
and rarely in the other documents.
Treat each source file as a document, and the highest scoring words become name candidates.

### Tokenizing Source Code
The default tokenizer in ``TfidfVectorizer`` is built for prose.
Source code needs a little help:
identifiers like ``parseHTTPResponse`` or ``raw_body`` should be split into words,
and Python keywords and builtins (``def``, ``return``, ``self``, ``len``) should be ignored
since every file has them.

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


print(tokenize("def parseHTTPResponse(self, raw_body): return None"))
print(tokenize("class TarInfo:  # holds the tar header"))
```
```
['parse', 'httpresponse', 'raw', 'body']
['tar', 'info', 'holds', 'tar', 'header']
```

``IDENTIFIER`` only matches letters, so underscores and digits split ``snake_case`` names for free.
``CAMEL`` splits on a lowercase letter followed by an uppercase letter.
Comments and docstrings are kept on purpose; they often contain the best description of the code.

### Sanity Check on the Standard Library
Before trusting this on my own code, I want to see if it can recover names that are already good.
The Python standard library is a nice test set since each module is already well named.
(The results below are from Python 3.12; other versions will shift slightly.)

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

To make it a real test, add a mystery file.
Here's a ``utils.py`` that I definitely did not name well:

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

### Fitting TF-IDF
Pass the custom ``tokenize`` function to ``TfidfVectorizer``.
``token_pattern=None`` silences a warning about the unused default pattern,
and ``lowercase=False`` skips a step the tokenizer already handles.
``sublinear_tf=True`` uses ``1 + log(tf)`` so a word that appears 200 times doesn't completely drown out one that appears 20 times.

```python
import pandas as pd
from sklearn.feature_extraction.text import TfidfVectorizer

vectorizer = TfidfVectorizer(
    tokenizer=tokenize,
    lowercase=False,
    token_pattern=None,
    sublinear_tf=True,
)
X = vectorizer.fit_transform(docs.values())
print(X.shape)

scores = pd.DataFrame(
    X.toarray(),
    index=list(docs),
    columns=vectorizer.get_feature_names_out(),
)

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
``calendar``, ``gzip``, ``smtp``, ``tar``, ``heap``, and ``uuid`` all show up in their own top five.

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
Treat the output as a list of suggestions.
It still takes a human to pick the name, but it's a lot easier to pick from a list than a blank page.

### Naming a Package
The same idea works one level up.
Join every ``.py`` file in a directory into one document, and each document now represents a package:

```python
PACKAGES = ["json", "email", "http", "logging", "sqlite3", "unittest", "asyncio", "importlib"]
pkg_docs = {
    pkg: "\n".join(path.read_text() for path in sorted((STDLIB / pkg).rglob("*.py")))
    for pkg in PACKAGES
}

pkg_vectorizer = TfidfVectorizer(
    tokenizer=tokenize,
    lowercase=False,
    token_pattern=None,
    sublinear_tf=True,
)
X_pkg = pkg_vectorizer.fit_transform(pkg_docs.values())
pkg_scores = pd.DataFrame(
    X_pkg.toarray(),
    index=list(pkg_docs),
    columns=pkg_vectorizer.get_feature_names_out(),
)

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

Packages are noisier than single files since they cover more ground.
Still, ``json``, ``sqlite``, ``mock``/``suite``, and ``loader`` point in the right direction.

### Running It on Your Own Code
Swap the standard library for your project.
Each file in the project is a document, which means the IDF part is computed against *your* codebase.
Words that appear everywhere in your project (your domain, your company name) get pushed down,
and the words specific to each file float up.

```python
PROJECT = Path("src/my_project")
docs = {str(path.relative_to(PROJECT)): path.read_text() for path in PROJECT.rglob("*.py")}
```

A few tips:
- Add project-wide words (the package name, ``logger``, ``config``) to ``STOP_WORDS`` if they crowd the results.
- Very small files don't have much signal. Use the results as a starting point.
- If two files share the same top terms, they might belong together.

### Conclusion
TF-IDF scores a word by how often it appears in one document and how rare it is across the rest.
Point it at source code, with a tokenizer that understands identifiers,
and the top terms describe what each file actually does.
It won't name your code for you, but it's a quick way to get out of ``utils.py``.

### Further Reading
- [``TfidfVectorizer``](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html){:target="_blank"}
- [TF-IDF (Wikipedia)](https://en.wikipedia.org/wiki/Tf%E2%80%93idf){:target="_blank"}
