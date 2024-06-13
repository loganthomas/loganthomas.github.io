---
layout: post
title:  "PyTorch Recipe: Inspecting Model Parameters"
date:   2024-06-10 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: deep-learning pytorch
---

## Inspecting Model Parameters

### What you will learn
- How to inspect a model's parameters using ``.parameters()`` and ``.named_parameters()``
- How to collect the trainable parameters of a model
- How to use the ``torchinfo`` package (formerly ``torch-summary``) to print a model summary

### Overview
When building neural networks, it's helpful to be able to inspect
parameters (model weights) at intermediate stages of development.

This can help inform model architecture decisions, like how many
neurons to put in a proceeding layer.
Or, it can be used for debugging purposes to ensure each model's layer
has the anticipated number of weights.

### Inspecting Parameters of a Simple Neural Network
Let's start with a simple example:

```python
from torch import nn


class NeuralNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        self.flatten = nn.Flatten()
        self.linear_relu_stack = nn.Sequential(
            nn.Linear(28 * 28, 512),
            nn.ReLU(),
            nn.Linear(512, 512),
            nn.ReLU(),
            nn.Linear(512, 10),
        )

    def forward(self, x):
        x = self.flatten(x)
        logits = self.linear_relu_stack(x)
        return logits


model = NeuralNetwork()
print(model)
```
```
NeuralNetwork(
  (flatten): Flatten(start_dim=1, end_dim=-1)
  (linear_relu_stack): Sequential(
    (0): Linear(in_features=784, out_features=512, bias=True)
    (1): ReLU()
    (2): Linear(in_features=512, out_features=512, bias=True)
    (3): ReLU()
    (4): Linear(in_features=512, out_features=10, bias=True)
  )
)
```

Layers inside a neural network are parameterized, i.e.
have associated weights and biases that are optimized during training.
Subclassing ``nn.Module`` automatically tracks all fields defined
inside a model object, and makes all parameters accessible using a
model’s ``parameters()`` or ``named_parameters()`` methods.

To inspect the shape of the parameter's associated with each layer in the model,
use ``model.parameters()``:

```python
print([param.shape for param in model.parameters()])
```
```
[torch.Size([512, 784]),
 torch.Size([512]),
 torch.Size([512, 512]),
 torch.Size([512]),
 torch.Size([10, 512]),
 torch.Size([10])]
```

Sometimes, it's more helpful to be able to have a name associated with
the parameters of each layer. Use ``model.named_parameters()`` to access
the parameter name in addition to the shape:

```python
for name, param in model.named_parameters():
    print(name, param.shape)
```
```
linear_relu_stack.0.weight torch.Size([512, 784])
linear_relu_stack.0.bias torch.Size([512])
linear_relu_stack.2.weight torch.Size([512, 512])
linear_relu_stack.2.bias torch.Size([512])
linear_relu_stack.4.weight torch.Size([10, 512])
linear_relu_stack.4.bias torch.Size([10])
```

Notice that the parameters are collected from the ``nn.Linear`` modules
specified in the network. Because the default behavior for ``nn.Linear``
is to include a bias term, the output shows both a ``weight`` and ``bias``
parameter for each of the ``nn.Linear`` modules.

The shape of these parameters relate to the input shape (``in_features``)
and output shape (``out_features``) specified in each of the model's layers.

Take for example the first ``nn.Linear(28*28, 512)`` module specified:

```python
layer = nn.Linear(28 * 28, 512)

for name, param in layer.named_parameters():
    print(name, param.size())
```
```
weight torch.Size([512, 784])
bias torch.Size([512])
```

The first line from the printed ``model.named_parameters()`` section
(``linear_relu_stack.0.weight torch.Size([512, 784])``) specifies
the ``weight`` associated with this layer.
The second line from the printed ``model.named_parameters()`` section
(``linear_relu_stack.0.bias torch.Size([512])``) specifies
the ``bias`` associated with this layer.

The printed statements using ``.named_parameters()``
are *not* meant to report the original shapes of the model's **layers**
but the shape of the **weights** (and/or **biases**) of the **parameters of the layers**.

This can cause confusion for new practitioners since the shape of the weights
seem to invert the input shape and output shape specified for the Linear layers.
These weights will be **transposed** during the matrix
multiplication process when the model makes a prediction (as specified in the [``nn.Linear``](https://pytorch.org/docs/stable/generated/torch.nn.Linear.html)
docstring).

There is also a helpful ``.numel()`` method that can be used to gather
the number of elements that are in each model parameter:

```python
for name, param in model.named_parameters():
    print(f"{name=}, {param.size()=}, {param.numel()=}")
```
```
name='linear_relu_stack.0.weight', param.size()=torch.Size([512, 784]), param.numel()=401408
name='linear_relu_stack.0.bias', param.size()=torch.Size([512]), param.numel()=512
name='linear_relu_stack.2.weight', param.size()=torch.Size([512, 512]), param.numel()=262144
name='linear_relu_stack.2.bias', param.size()=torch.Size([512]), param.numel()=512
name='linear_relu_stack.4.weight', param.size()=torch.Size([10, 512]), param.numel()=5120
name='linear_relu_stack.4.bias', param.size()=torch.Size([10]), param.numel()=10
```

The number of elements for each parameter is calculated by taking
the product of the entries of the Size tensor.
The ``.numel()`` can be used to find all the parameters in a model by taking
the sum across all the layer parameters:

```python
print(f"Total model params: {sum(p.numel() for p in model.parameters()):,}")
```
```
Total model params: 669,706
```

Sometimes, only the *trainable* parameters are of interest.
Use the ``requires_grad`` attribute to collect only those parameters
that require a gradient to be computed (i.e. those parameters that will be optimized during model training):

```python
print(f"Total model trainable params: {sum(p.numel() for p in model.parameters() if p.requires_grad):,}")
```
```
Total model trainable params: 669,706
```

Since all the model weights currently require a gradient, the number
of trainable parameters are the same as the total number of model
parameters. Simply for educational purposes, parameters can be frozen
to show a difference in count. Below, the first linear layer's ``weight`` parameters are frozen
by setting ``requires_grad=False`` which will result in the trainable
parameters count having 401,408 less parameters.

```python
for name, param in model.named_parameters():
    if name == "linear_relu_stack.0.weight":
        param.requires_grad = False
    print(f"{name=}, {param.size()=}, {param.numel()=}, {param.requires_grad=}")
print(f"Total model trainable params: {sum(p.numel() for p in model.parameters() if p.requires_grad):,}")
```
```
name='linear_relu_stack.0.weight', param.size()=torch.Size([512, 784]), param.numel()=401408, param.requires_grad=False
name='linear_relu_stack.0.bias', param.size()=torch.Size([512]), param.numel()=512, param.requires_grad=True
name='linear_relu_stack.2.weight', param.size()=torch.Size([512, 512]), param.numel()=262144, param.requires_grad=True
name='linear_relu_stack.2.bias', param.size()=torch.Size([512]), param.numel()=512, param.requires_grad=True
name='linear_relu_stack.4.weight', param.size()=torch.Size([10, 512]), param.numel()=5120, param.requires_grad=True
name='linear_relu_stack.4.bias', param.size()=torch.Size([10]), param.numel()=10, param.requires_grad=True
Total model trainable params: 268,298
```

### Inspecting Parameters of a Convolutional Neural Network
These techniques also work for Convolutional Neural Networks:

```python
class CNN(nn.Module):
    def __init__(self):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(in_channels=1, out_channels=32, kernel_size=3),
            nn.ReLU(inplace=True),
            nn.Conv2d(in_channels=32, out_channels=64, kernel_size=3),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(kernel_size=(2, 2)),
        )
        self.classifier = nn.Sequential(
            nn.Flatten(),
            nn.Linear(64 * 12 * 12, 128),
            nn.ReLU(inplace=True),
            nn.Linear(128, 10),
        )

    def forward(self, x):
        x = self.features(x)
        x = self.classifier(x)
        return x


cnn_model = CNN()
print(cnn_model)
print("-" * 72)
for name, param in cnn_model.named_parameters():
    print(f"{name=}, {param.size()=}, {param.numel()=}, {param.requires_grad=}")
print("-" * 72)
print(f"Total model trainable params: {sum(p.numel() for p in cnn_model.parameters() if p.requires_grad):,}")
```
```
CNN(
  (features): Sequential(
    (0): Conv2d(1, 32, kernel_size=(3, 3), stride=(1, 1))
    (1): ReLU(inplace=True)
    (2): Conv2d(32, 64, kernel_size=(3, 3), stride=(1, 1))
    (3): ReLU(inplace=True)
    (4): MaxPool2d(kernel_size=(2, 2), stride=(2, 2), padding=0, dilation=1, ceil_mode=False)
  )
  (classifier): Sequential(
    (0): Flatten(start_dim=1, end_dim=-1)
    (1): Linear(in_features=9216, out_features=128, bias=True)
    (2): ReLU(inplace=True)
    (3): Linear(in_features=128, out_features=10, bias=True)
  )
)
------------------------------------------------------------------------
name='features.0.weight', param.size()=torch.Size([32, 1, 3, 3]), param.numel()=288, param.requires_grad=True
name='features.0.bias', param.size()=torch.Size([32]), param.numel()=32, param.requires_grad=True
name='features.2.weight', param.size()=torch.Size([64, 32, 3, 3]), param.numel()=18432, param.requires_grad=True
name='features.2.bias', param.size()=torch.Size([64]), param.numel()=64, param.requires_grad=True
name='classifier.1.weight', param.size()=torch.Size([128, 9216]), param.numel()=1179648, param.requires_grad=True
name='classifier.1.bias', param.size()=torch.Size([128]), param.numel()=128, param.requires_grad=True
name='classifier.3.weight', param.size()=torch.Size([10, 128]), param.numel()=1280, param.requires_grad=True
name='classifier.3.bias', param.size()=torch.Size([10]), param.numel()=10, param.requires_grad=True
------------------------------------------------------------------------
Total model trainable params: 1,199,882
```

As with the simple network example above, the number of elements per parameter
is the product of the parameter size:

```python
import numpy as np

for name, param in cnn_model.named_parameters():
    print(f"{name=}, {param.size()=}, {np.prod(param.size())=} == {param.numel()=}")
```
```
name='features.0.weight', param.size()=torch.Size([32, 1, 3, 3]), np.prod(param.size())=288 == param.numel()=288
name='features.0.bias', param.size()=torch.Size([32]), np.prod(param.size())=32 == param.numel()=32
name='features.2.weight', param.size()=torch.Size([64, 32, 3, 3]), np.prod(param.size())=18432 == param.numel()=18432
name='features.2.bias', param.size()=torch.Size([64]), np.prod(param.size())=64 == param.numel()=64
name='classifier.1.weight', param.size()=torch.Size([128, 9216]), np.prod(param.size())=1179648 == param.numel()=1179648
name='classifier.1.bias', param.size()=torch.Size([128]), np.prod(param.size())=128 == param.numel()=128
name='classifier.3.weight', param.size()=torch.Size([10, 128]), np.prod(param.size())=1280 == param.numel()=1280
name='classifier.3.bias', param.size()=torch.Size([10]), np.prod(param.size())=10 == param.numel()=10
```

For a more robust approach, consider using the [``torchinfo`` package](https://github.com/TylerYep/torchinfo) (formerly ``torch-summary``).
This package provides information complementary to what is provided by ``print(model)`` in PyTorch,
similar to Tensorflow's ``model.summary()`` API to view the visualization of the model.

Notice that the trainable parameters reported by ``torchinfo`` matches
the manually gathered trainable parameters.

```python
import torchinfo

# If running from a notebook, use print(torchinfo.summary(model))
torchinfo.summary(model)
print("-" * 72)
print(f"Manually gathered model trainable params: {sum(p.numel() for p in model.parameters() if p.requires_grad):,}")
```
```
=================================================================
Layer (type:depth-idx)                   Param #
=================================================================
NeuralNetwork                            --
├─Flatten: 1-1                           --
├─Sequential: 1-2                        --
│    └─Linear: 2-1                       401,920
│    └─ReLU: 2-2                         --
│    └─Linear: 2-3                       262,656
│    └─ReLU: 2-4                         --
│    └─Linear: 2-5                       5,130
=================================================================
Total params: 669,706
Trainable params: 268,298
Non-trainable params: 401,408
=================================================================
------------------------------------------------------------------------
Manually gathered model trainable params: 268,298
```

There is one minor, but important, difference in the way ``torchinfo`` reports the number of parameters per layer.
Notice that the ``weight`` and ``bias`` parameter counts are **combined**
to report on the *total* number of parameters per layer.

For example, the first linear layer of the ``model`` created in the
"Inspecting Parameters of a Simple Neural Network" section has a
``weight`` parameter with ``401,408`` elements and a ``bias`` parameter
with ``512``. Combining these two yields a total
of ``401,920`` (``401,408+512``) parameters for the layer -- which is
equivalent to what the ``torchinfo`` summary showed.

A similar report can be generated manually by summing parameters per layer:

```python
from collections import defaultdict

layer_params = defaultdict(int)

for name, param in model.named_parameters():
    # combine weight and bias together using layer name
    # linear_relu_stack.0 = linear_relu_stack.0.weight + linear_relu_stack.bias
    layer_params[name.rsplit(".", 1)[0]] += param.numel()

for name, total_params in layer_params.items():
    print(f"{name=} {total_params=:,}")
```
```
name='linear_relu_stack.0' total_params=401,920
name='linear_relu_stack.2' total_params=262,656
name='linear_relu_stack.4' total_params=5,130
```

These approaches works for the Convolutional Neural Network as well:

```python
# If running from a notebook, use print(torchinfo.summary(model))
torchinfo.summary(cnn_model)
print("-" * 72)
print(
    f"Manually gathered model trainable params: {sum(p.numel() for p in cnn_model.parameters() if p.requires_grad):,}"
)
print("-" * 72)
print("Manually generated total number of parameters per layer:")
cnn_layer_params = defaultdict(int)

for name, param in cnn_model.named_parameters():
    cnn_layer_params[name.rsplit(".", 1)[0]] += param.numel()

for name, total_params in cnn_layer_params.items():
    print(f"{name=} {total_params=:,}")
```
```
=================================================================
Layer (type:depth-idx)                   Param #
=================================================================
CNN                                      --
├─Sequential: 1-1                        --
│    └─Conv2d: 2-1                       320
│    └─ReLU: 2-2                         --
│    └─Conv2d: 2-3                       18,496
│    └─ReLU: 2-4                         --
│    └─MaxPool2d: 2-5                    --
├─Sequential: 1-2                        --
│    └─Flatten: 2-6                      --
│    └─Linear: 2-7                       1,179,776
│    └─ReLU: 2-8                         --
│    └─Linear: 2-9                       1,290
=================================================================
Total params: 1,199,882
Trainable params: 1,199,882
Non-trainable params: 0
=================================================================
------------------------------------------------------------------------
Manually gathered model trainable params: 1,199,882
------------------------------------------------------------------------
Manually generated total number of parameters per layer:
name='features.0' total_params=320
name='features.2' total_params=18,496
name='classifier.1' total_params=1,179,776
name='classifier.3' total_params=1,290
```

### Conclusion
Layers inside a neural network have associated weights and biases
that are optimized during training. These parameters (model weights)
are made accessible using a model’s ``parameters()`` or ``named_parameters()``
methods. Interacting with these parameters can help inform model
architecture decisions or support model debugging.

### Further Reading
- [``torchinfo``](https://github.com/TylerYep/torchinfo): provides information complementary to what is provided by ``print(model)`` in PyTorch, similar to Tensorflow's model.summary() API.
