---
layout: post
title:  "Python Recipe: Dotted Attribute Access with SimpleNamespace and Bunch"
date:   2026-10-03 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: python
---

## Dotted Attribute Access with SimpleNamespace and Bunch

### What you will learn
- How to create a quick object with dotted access using ``types.SimpleNamespace``
- How scikit-learn's ``Bunch`` supports both ``obj.key`` and ``obj["key"]``
- How to write a minimal ``Bunch`` yourself
- How to load nested JSON into dotted objects

### Overview
Typing ``config["model"]["lr"]`` gets old fast.
Sometimes I just want ``config.model.lr``, without writing a whole class or ``dataclass`` for a throwaway object.
If you've used scikit-learn, you've seen this with ``load_iris().data``.
Here are the options I keep coming back to.

### types.SimpleNamespace
``SimpleNamespace`` is in the standard library and is the quickest option.
Pass keyword arguments and get an object with those attributes:

```python
from types import SimpleNamespace

config = SimpleNamespace(lr=0.001, epochs=10, batch_size=64)
print(config)
print(config.lr)
```
```
namespace(lr=0.001, epochs=10, batch_size=64)
0.001
```

Attributes can be updated or added after the fact:

```python
config.epochs = 20
config.optimizer = "adam"
print(config)
```
```
namespace(lr=0.001, epochs=20, batch_size=64, optimizer='adam')
```

To go from a ``dict`` to a namespace, unpack it.
To go back, use ``vars()``:

```python
params = {"lr": 0.001, "epochs": 10}
config = SimpleNamespace(**params)
print(vars(config))
```
```
{'lr': 0.001, 'epochs': 10}
```

``SimpleNamespace`` also gives you a readable ``repr`` and equality comparison for free,
which a bare ``object()`` subclass does not.

### scikit-learn's Bunch
scikit-learn's dataset loaders return a ``Bunch``:

```python
from sklearn.datasets import load_iris

iris = load_iris()
print(type(iris))
print(iris.keys())
print(iris.target_names)
print(iris["target_names"])
```
```
<class 'sklearn.utils._bunch.Bunch'>
dict_keys(['data', 'target', 'frame', 'target_names', 'DESCR', 'feature_names', 'filename', 'data_module'])
['setosa' 'versicolor' 'virginica']
['setosa' 'versicolor' 'virginica']
```

A ``Bunch`` is a ``dict`` subclass, so both dotted access and key access work.
That is the main difference from ``SimpleNamespace``:
you keep all the ``dict`` methods (``.keys()``, ``.items()``, ``**`` unpacking, JSON serialization).

It can be imported and used directly:

```python
from sklearn.utils import Bunch

b = Bunch(lr=0.001, epochs=10)
b.batch_size = 64
print(b)
print(b["batch_size"], b.epochs)
```
```
{'lr': 0.001, 'epochs': 10, 'batch_size': 64}
64 10
```

### Writing Your Own Bunch
If scikit-learn isn't already a dependency, it's not worth adding one for this.
A minimal version is a few lines:

```python
class Bunch(dict):
    def __init__(self, **kwargs):
        super().__init__(kwargs)

    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError(key) from None

    def __setattr__(self, key, value):
        self[key] = value


b = Bunch(lr=0.001, epochs=10)
b.batch_size = 64
print(b)
print(b.lr, b["batch_size"])
```
```
{'lr': 0.001, 'epochs': 10, 'batch_size': 64}
0.001 64
```

Raising ``AttributeError`` (not ``KeyError``) for a missing key matters.
Tools like ``getattr()`` with a default and ``hasattr()`` expect it:

```python
print(getattr(b, "momentum", 0.9))
```
```
0.9
```

Keep in mind that keys which collide with ``dict`` methods (like ``b.items`` or ``b.keys``)
will return the method, not your value.
Use ``b["items"]`` for those.

### Nested JSON with Dotted Access
``json.loads()`` accepts an ``object_hook`` that is called on every decoded ``dict``.
Pointing it at ``SimpleNamespace`` converts the whole nested structure:

```python
import json

payload = '{"model": {"name": "resnet", "layers": 50}, "train": {"lr": 0.001}}'
cfg = json.loads(payload, object_hook=lambda d: SimpleNamespace(**d))

print(cfg.model.name, cfg.train.lr)
print(cfg)
```
```
resnet 0.001
namespace(model=namespace(name='resnet', layers=50), train=namespace(lr=0.001))
```

### argparse.Namespace
If you've parsed command line arguments, you've already used one of these.
``argparse`` returns a ``Namespace`` that works the same way:

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--lr", type=float, default=0.001)
args = parser.parse_args(["--lr", "0.01"])
print(args, args.lr)
```
```
Namespace(lr=0.01) 0.01
```

### Conclusion
For a quick object with dotted access, use ``types.SimpleNamespace``.
If you also need it to behave like a ``dict``, use a ``Bunch``
(either scikit-learn's or the few-line version above).
Once the structure is fixed and shared across a codebase,
that's the point to graduate to a ``dataclass`` or ``NamedTuple``.

### Further Reading
- [``types.SimpleNamespace``](https://docs.python.org/3/library/types.html#types.SimpleNamespace){:target="_blank"}
- [``sklearn.utils.Bunch``](https://scikit-learn.org/stable/modules/generated/sklearn.utils.Bunch.html){:target="_blank"}
