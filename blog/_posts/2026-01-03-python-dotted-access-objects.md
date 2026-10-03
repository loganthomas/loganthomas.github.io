---
layout: post
title:  "Connect the Dots in Your dict"
date:   2026-01-03 08:00:00 -0600
author: Logan Thomas
categories: blog
tags: python
---

Dictionaries are great until you're three keys deep in square brackets.
Typing ``config["model"]["lr"]`` gets old fast.
Sometimes I just want ``config.model.lr``.
This post covers simple ways to get dotted access on a dictionary:
``types.SimpleNamespace``, scikit-learn's ``Bunch``,
a custom ``DotDict`` that subclasses the built-in ``dict``,
and loading nested JSON straight into dotted objects.
At the end there's a table and a quick guide for picking one.

The textbook answer is a ``dataclass``:

```python
from dataclasses import dataclass


@dataclass
class ModelConfig:
    name: str
    lr: float


@dataclass
class Config:
    model: ModelConfig
    epochs: int


config = Config(model=ModelConfig(name="resnet", lr=0.001), epochs=10)
print(config.model.lr)
```

```
0.001
```

That's the right tool when the structure is fixed and shared across a codebase.
For a throwaway object, it's a lot of ceremony.
Every field has to be declared up front,
so adding a key means editing a class definition,
and an unexpected key in the input is a ``TypeError``:

```python
Config(
    model=ModelConfig(name="resnet", lr=0.001),
    epochs=10,
    seed=42,
)
```

```
TypeError: Config.__init__() got an unexpected keyword argument 'seed'
```

Sometimes I want something lighter.
I did some digging and found a few options.

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
print(config)
print(vars(config))
```

```
namespace(lr=0.001, epochs=10)
{'lr': 0.001, 'epochs': 10}
```

``SimpleNamespace`` also gives you a readable ``repr`` and equality comparison for free:

```python
other = SimpleNamespace(lr=0.001, epochs=10)
print(repr(other))
print(config == other)
```

```
namespace(lr=0.001, epochs=10)
True
```

**Aside:** if you've ever parsed command line arguments,
you've already used this without knowing it.
``argparse`` returns a ``Namespace`` that works the same way:

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--lr", type=float, default=0.001)
args = parser.parse_args(["--lr", "0.01"])
print(args)
print(args.lr)
```

```
Namespace(lr=0.01)
0.01
```

**Advantages**
- Standard library, no dependencies
- Readable ``repr`` and ``==`` comparison out of the box
- Easy round-trip with ``dict`` via ``**`` unpacking and ``vars()``

**Disadvantages**
- Not a ``dict``: no ``.keys()``, ``.items()``, ``in``, or ``obj["key"]``
- Not JSON-serializable without ``vars()`` first
- Nested ``dict`` values stay ``dict``s unless you convert them yourself

### scikit-learn's Bunch
scikit-learn's dataset loaders return a ``Bunch``:

```python
from sklearn.datasets import load_iris

iris = load_iris()
print(type(iris))
print(iris.target_names)
print(iris["target_names"])
```

```
<class 'sklearn.utils._bunch.Bunch'>
['setosa' 'versicolor' 'virginica']
['setosa' 'versicolor' 'virginica']
```

A ``Bunch`` is a ``dict`` subclass, so both dotted access and key access work.
That is the main difference from ``SimpleNamespace``:
you keep all the ``dict`` methods (``.keys()``, ``.items()``, ``**`` unpacking, JSON serialization).

It can be imported and used directly:

```python
from sklearn.utils import Bunch

config = Bunch(lr=0.001, epochs=10)
config.batch_size = 64
print(config)
print(config.epochs, config["batch_size"])
```

```
{'lr': 0.001, 'epochs': 10, 'batch_size': 64}
10 64
```

**Advantages**
- Both ``obj.key`` and ``obj["key"]`` work
- Full ``dict`` API, so it plugs into ``json.dumps()``, ``**`` unpacking, and anything expecting a mapping
- Keys show up in tab completion
- Familiar to anyone who has used scikit-learn's datasets

**Disadvantages**
- Pulls in scikit-learn as a dependency for a tiny utility
- Keys that collide with ``dict`` methods (``keys``, ``items``, ``update``) are shadowed by the method
- Nested ``dict`` values are not converted to ``Bunch``

### Writing Your Own DotDict
If scikit-learn isn't already a dependency, it's not worth adding one for this.
A minimal version is a few lines:

```python
class DotDict(dict):
    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError(key) from None

    def __setattr__(self, key, value):
        self[key] = value


config = DotDict(lr=0.001, epochs=10)
config.batch_size = 64
print(config)
print(config.lr, config["batch_size"])
```

```
{'lr': 0.001, 'epochs': 10, 'batch_size': 64}
0.001 64
```

**Why it works**

When you type ``config.lr``, Python first looks for a normal attribute named ``lr``.
It checks the class and the object itself.
But ``lr`` is a key, not an attribute, so that search fails.
Python then calls ``__getattr__`` with the name as a string,
and ``DotDict`` looks up that key.

Setting a value works a little differently.
``config.batch_size = 64`` always calls ``__setattr__``,
so ``DotDict`` stores the value as a key.

When a key is missing, ``__getattr__`` raises ``AttributeError``, not ``KeyError``.
That's what ``getattr()`` and ``hasattr()`` expect:

```python
print(getattr(config, "momentum", 0.9))
```

```
0.9
```

Since ``__getattr__`` only runs when the normal search fails,
``dict`` methods win over keys.
``config.items`` gives you the method, not your value.
Use ``config["items"]`` for keys like that.

**Advantages**
- No dependency
- Same dotted and key access as scikit-learn's ``Bunch``
- Easy to extend (e.g., recursive conversion of nested ``dict``s)

**Disadvantages**
- One more piece of code you own, test, and maintain
- Same ``dict`` method-name collisions as scikit-learn's version
- This minimal version lacks niceties like tab completion (``__dir__``)

### The Nested dict Problem
None of the options so far convert nested ``dict``s.
Only the top level gets dotted access:

```python
data = {"model": {"name": "resnet", "layers": 50}, "train": {"lr": 0.001}}
config = SimpleNamespace(**data)
print(config.model)
print(config.model.name)
```

```
{'name': 'resnet', 'layers': 50}
AttributeError: 'dict' object has no attribute 'name'
```

``config.model`` is still a plain ``dict``,
so ``config.model.name`` fails.
``Bunch`` and ``DotDict`` work the same way.
To fix that, you have to convert each level yourself, recursively:

```python
def to_namespace(obj):
    if isinstance(obj, dict):
        return SimpleNamespace(**{k: to_namespace(v) for k, v in obj.items()})
    if isinstance(obj, list):
        return [to_namespace(item) for item in obj]
    return obj


config = to_namespace(data)
print(config.model.name, config.train.lr)
print(config)
```

```
resnet 0.001
namespace(model=namespace(name='resnet', layers=50), train=namespace(lr=0.001))
```

Or we could turn the ``dict`` into JSON and let ``json.loads()`` do the work.

### Nested JSON with Dotted Access
``json.loads()`` accepts an ``object_hook`` that is called on every decoded ``dict``.
Pointing it at ``SimpleNamespace`` converts the whole nested structure:

```python
import json

payload = '{"model": {"name": "resnet", "layers": 50}, "train": {"lr": 0.001}}'
config = json.loads(payload, object_hook=lambda d: SimpleNamespace(**d))
print(config.model.name, config.train.lr)
print(config)
```

```
resnet 0.001
namespace(model=namespace(name='resnet', layers=50), train=namespace(lr=0.001))
```

**Advantages**
- One line converts every level of nesting, including ``dict``s inside lists
- Works with any callable, so ``object_hook=DotDict`` gives nested ``DotDict``s instead

**Disadvantages**
- A ``dict`` you already have needs a ``json.dumps()`` round trip first,
  which turns tuples into lists and non-string keys into strings
- With ``SimpleNamespace`` as the hook, going back to JSON needs ``json.dumps(config, default=vars)``
- Keys that aren't valid identifiers (like ``"batch-size"``) can't be reached with a dot

### Summary

| Approach                        | ``obj.key`` | ``obj["key"]``                            | ``dict``&nbsp;API                         | Nested conversion                          | Dependency       |
|---------------------------------|:-----------:|:-----------------------------------------:|:-----------------------------------------:|:------------------------------------------:|------------------|
| ``types.SimpleNamespace``       | ✅          | ❌                                        | ❌                                        | ❌                                         | Standard library |
| ``sklearn.utils.Bunch``         | ✅          | ✅                                        | ✅                                        | ❌                                         | scikit-learn     |
| Custom ``DotDict``              | ✅          | ✅                                        | ✅                                        | ❌<sup style="position: absolute">**</sup> | None             |
| ``json.loads(object_hook=...)`` | ✅          | ✅<sup style="position: absolute">*</sup> | ✅<sup style="position: absolute">*</sup> | ✅                                         | Standard library |

\* With ``object_hook=DotDict``.
``SimpleNamespace`` as the hook gives neither.

\*\* Not built in, but a small recursive function adds it (see The Nested dict Problem).

### When to Use Which
Most of the time, ``SimpleNamespace`` is the one to reach for.
It's in the standard library and covers the common case.
If you need it to act like a ``dict``, write the ``DotDict``.
Here's a bit more detail:

- **Quick, flat, throwaway object**: ``types.SimpleNamespace``.
- **Needs to behave like a ``dict`` and scikit-learn is already installed**: ``sklearn.utils.Bunch``.
- **Needs to behave like a ``dict`` without adding a dependency**: the few-line ``DotDict``.
- **Nested config loaded from JSON**: ``json.loads()`` with an ``object_hook``.
- **Structure is fixed and shared across a codebase**: graduate to a ``dataclass`` or ``NamedTuple`` for type hints, defaults, and optional immutability.

### Further Reading
- [``types.SimpleNamespace``](https://docs.python.org/3/library/types.html#types.SimpleNamespace){:target="_blank"}
- [``sklearn.utils.Bunch``](https://scikit-learn.org/stable/modules/generated/sklearn.utils.Bunch.html){:target="_blank"}
- [``json.loads()`` and ``object_hook``](https://docs.python.org/3/library/json.html#json.load){:target="_blank"}
- [``argparse.Namespace``](https://docs.python.org/3/library/argparse.html#argparse.Namespace){:target="_blank"}
