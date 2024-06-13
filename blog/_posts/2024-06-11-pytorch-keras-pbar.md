---
layout: post
title:  "PyTorch Recipe: Implementing a Keras Progress Bar"
date:   2024-06-11 08:00:00 -0500
author: Logan Thomas
categories: blog
tags: deep-learning pytorch
---

## Implementing a Keras Progress Bar

### What you will learn
- How to implement Keras's model training progress bar in PyTorch

### Overview
Keras implements a progress bar that shows a nice snapshot of what is occurring during model training.
This offers a more visually appealing command line report and
is less burdensome than a developer having to write code to gather important metrics.

Adding a Keras progress bar only takes a few lines of code and is a nice addition to any training/test loop.

### Traditional Approach
Let's start with a simple example borrowing from the below [Learn the Basics Tutorials](https://pytorch.org/tutorials/beginner/basics/intro.html):
- [Datasets & DataLoaders](https://pytorch.org/tutorials/beginner/basics/data_tutorial.html)
- [Build the Neural Network](https://pytorch.org/tutorials/beginner/basics/buildmodel_tutorial.html)
- [Optimizing Model Parameters](https://pytorch.org/tutorials/beginner/basics/optimization_tutorial.html)

```python
import torch
from torch import nn
from torch.utils.data import DataLoader
from torchvision import datasets
from torchvision.transforms import ToTensor

BATCH_SIZE = 64

training_data = datasets.FashionMNIST(root="data", train=True, download=True, transform=ToTensor())

test_data = datasets.FashionMNIST(root="data", train=False, download=True, transform=ToTensor())

train_dataloader = DataLoader(training_data, batch_size=BATCH_SIZE)
test_dataloader = DataLoader(test_data, batch_size=BATCH_SIZE)


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

#### Define a Traditional Training Loop
```python
def train_loop(dataloader, model, batch_size, loss_fn, optimizer):
    size = len(dataloader.dataset)
    # Set the model to training mode
    # Important for batch normalization and dropout layers
    # Unnecessary in this situation but added for best practices
    model.train()
    for batch, (X, y) in enumerate(dataloader):
        # Compute prediction and loss
        pred = model(X)
        loss = loss_fn(pred, y)

        # Backpropagation
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()

        if batch % 100 == 0:
            loss, current = loss.item(), batch * batch_size + len(X)
            print(f"loss: {loss:>7f}  [{current:>5d}/{size:>5d}]")
```

#### Define a Traditional Testing Loop
```python
def test_loop(dataloader, model, loss_fn):
    # Set the model to evaluation mode
    # Important for batch normalization and dropout layers
    # Unnecessary in this situation but added for best practices
    model.eval()
    size = len(dataloader.dataset)
    num_batches = len(dataloader)
    test_loss, correct = 0, 0

    # Evaluating the model with torch.no_grad()
    # ensures that no gradients are computed during test mode
    # also serves to reduce unnecessary gradient computations
    # and memory usage for tensors with requires_grad=True
    with torch.no_grad():
        for X, y in dataloader:
            pred = model(X)
            test_loss += loss_fn(pred, y).item()
            correct += (pred.argmax(1) == y).type(torch.float).sum().item()

    test_loss /= num_batches
    correct /= size
    print(f"Test Error: \n Accuracy: {(100*correct):>0.1f}%, Avg loss: {test_loss:>8f} \n")
```

### Train the Model

```python
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.SGD(model.parameters(), lr=1e-3)

epochs = 10
batch_size = BATCH_SIZE
for t in range(epochs):
    print(f"Epoch {t+1}\n-------------------------------")
    train_loop(train_dataloader, model, batch_size, loss_fn, optimizer)
    test_loop(test_dataloader, model, loss_fn)
print("Done!")
```
```
Epoch 1
-------------------------------
loss: 2.299914  [   64/60000]
loss: 2.298036  [ 6464/60000]
loss: 2.276126  [12864/60000]
loss: 2.267775  [19264/60000]
loss: 2.249449  [25664/60000]
loss: 2.211145  [32064/60000]
loss: 2.231255  [38464/60000]
loss: 2.191056  [44864/60000]
loss: 2.174907  [51264/60000]
loss: 2.138854  [57664/60000]
Test Error:
 Accuracy: 38.1%, Avg loss: 2.141976

Epoch 2
-------------------------------
loss: 2.158404  [   64/60000]
loss: 2.150520  [ 6464/60000]
loss: 2.085808  [12864/60000]
loss: 2.093486  [19264/60000]
loss: 2.037194  [25664/60000]
loss: 1.976014  [32064/60000]
loss: 2.012927  [38464/60000]
loss: 1.925277  [44864/60000]
loss: 1.924248  [51264/60000]
loss: 1.845655  [57664/60000]
Test Error:
 Accuracy: 49.5%, Avg loss: 1.848806

Epoch 3
-------------------------------
loss: 1.892331  [   64/60000]
loss: 1.859852  [ 6464/60000]
loss: 1.735108  [12864/60000]
loss: 1.771755  [19264/60000]
loss: 1.657763  [25664/60000]
loss: 1.619588  [32064/60000]
loss: 1.649848  [38464/60000]
loss: 1.547130  [44864/60000]
loss: 1.573486  [51264/60000]
loss: 1.469762  [57664/60000]
Test Error:
 Accuracy: 59.6%, Avg loss: 1.487445

...
...
<clipped output>
...
...

Epoch 9
-------------------------------
loss: 0.879913  [   64/60000]
loss: 0.941636  [ 6464/60000]
loss: 0.724766  [12864/60000]
loss: 0.908513  [19264/60000]
loss: 0.808189  [25664/60000]
loss: 0.799041  [32064/60000]
loss: 0.880020  [38464/60000]
loss: 0.834604  [44864/60000]
loss: 0.849155  [51264/60000]
loss: 0.818514  [57664/60000]
Test Error:
 Accuracy: 69.5%, Avg loss: 0.818927

Epoch 10
-------------------------------
loss: 0.827772  [   64/60000]
loss: 0.901987  [ 6464/60000]
loss: 0.678783  [12864/60000]
loss: 0.872766  [19264/60000]
loss: 0.777142  [25664/60000]
loss: 0.760889  [32064/60000]
loss: 0.846925  [38464/60000]
loss: 0.809381  [44864/60000]
loss: 0.816719  [51264/60000]
loss: 0.789865  [57664/60000]
Test Error:
 Accuracy: 70.6%, Avg loss: 0.788775

Done!
```

### Setup a Keras Progress Bar
The same train loop and test loop can be performed using Keras's [``ProgBar`` class](https://keras.io/api/utils/python_utils/#progbar-class).
This can cut down on the code needed to report metrics during training and be more visual appealing.

#### Define a Training Loop with a Progress Bar
```python
import os

os.environ["KERAS_BACKEND"] = "torch"
import keras


def train_loop_pbar(dataloader, model, loss_fn, optimizer, epochs):
    # Move the epoch looping into the train loop
    # so epoch and pbar reported together
    for t in range(epochs):
        print(f"Epoch {t+1}/{epochs}")

        # Set the target of the pbar to
        # the number of iterations (batches) in the dataloader
        n_batches = len(dataloader)
        pbar = keras.utils.Progbar(target=n_batches)

        model.train()
        for batch, (X, y) in enumerate(dataloader):
            # Compute prediction and loss
            pred = model(X)
            loss = loss_fn(pred, y)
            correct = (pred.argmax(1) == y).type(torch.float).sum().item()
            correct /= len(y)

            # Backpropagation
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()

            # Update the pbar with each batch
            pbar.update(batch, values=[("loss", loss), ("acc", correct)])

            # Finish the progress bar with a final update
            # This ensures the progress bar reaches the end
            # (i.e. the target of n_batches is met on the last batch)
            if batch + 1 == n_batches:
                pbar.update(n_batches, values=None)
```

#### Define a Testing Loop with a Progress Bar
```python
def test_loop_pbar(dataloader, model, loss_fn):
    n_batches = len(dataloader)
    pbar = keras.utils.Progbar(target=n_batches)

    model.eval()
    with torch.no_grad():
        for batch, (X, y) in enumerate(dataloader):
            pred = model(X)
            loss = loss_fn(pred, y)
            correct = (pred.argmax(1) == y).type(torch.float).sum().item()
            correct /= len(y)

            pbar.update(batch, values=[("loss", loss), ("acc", correct)])

            if batch + 1 == n_batches:
                pbar.update(n_batches, values=None)
```

#### Train the Model with a Progress Bar

```python
# Re-instantiate model
model = NeuralNetwork()

# Train the model with progress bar
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.SGD(model.parameters(), lr=1e-3)

epochs = 10
train_loop_pbar(train_dataloader, model, loss_fn, optimizer, epochs)
```
```
Epoch 1/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 8s 8ms/step - loss: 2.2195 - acc: 0.4295
Epoch 2/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 7s 8ms/step - loss: 1.9854 - acc: 0.5649
Epoch 3/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 7s 8ms/step - loss: 1.6332 - acc: 0.5863
Epoch 4/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 7s 8ms/step - loss: 1.3331 - acc: 0.6315
Epoch 5/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 8s 9ms/step - loss: 1.1353 - acc: 0.6518
Epoch 6/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 9s 10ms/step - loss: 1.0048 - acc: 0.6651
Epoch 7/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 8s 9ms/step - loss: 0.9169 - acc: 0.6771
Epoch 8/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 9s 10ms/step - loss: 0.8559 - acc: 0.6902
Epoch 9/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 9s 10ms/step - loss: 0.8116 - acc: 0.7033
Epoch 10/10
938/938 ━━━━━━━━━━━━━━━━━━━━ 9s 9ms/step - loss: 0.7777 - acc: 0.7165
```
```python
# Evaluate the model with progress bar
test_loop_pbar(test_dataloader, model, loss_fn)
```
```
157/157 ━━━━━━━━━━━━━━━━━━━━ 2s 6ms/step - loss: 0.7776 - acc: 0.7116
```

### Including a Validation Dataset
The progress bar can handle validation datasets as well.
Start by splitting the existing training data into a new training set and validation set using an 80%-20% split.
That is, ``48,000`` new training and ``12,000`` validation datapoints.

```python
new_train_set, val_set = torch.utils.data.random_split(
    training_data, [int(len(training_data) * 0.8), int(len(training_data) * 0.2)]
)

new_train_dataloader = DataLoader(new_train_set, batch_size=BATCH_SIZE)
val_dataloader = DataLoader(val_set, batch_size=BATCH_SIZE)

print(len(train_dataloader), len(train_dataloader.dataset))
print()
print(len(new_train_dataloader), len(new_train_dataloader.dataset))
print(len(val_dataloader), len(val_dataloader.dataset))
```
```
938 60000

750 48000
188 12000
```

#### Define a Training Loop with a Progress Bar Including Validation Data
```python
def train_loop_pbar_with_val(train_dataloader, val_dataloader, model, loss_fn, optimizer, epochs):
    for t in range(epochs):
        print(f"Epoch {t+1}/{epochs}")

        n_batches = len(train_dataloader)
        pbar = keras.utils.Progbar(target=n_batches)

        # Train step
        model.train()
        for batch, (X, y) in enumerate(train_dataloader):
            # Compute prediction and loss
            pred = model(X)
            loss = loss_fn(pred, y)
            correct = (pred.argmax(1) == y).type(torch.float).sum().item()
            correct /= len(y)

            # Backpropagation
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()

            # Update the pbar with each batch
            pbar.update(batch, values=[("loss", loss), ("acc", correct)])

        # Validation step
        model.eval()
        val_loss, val_correct = 0, 0
        with torch.no_grad():
            for X, y in val_dataloader:
                val_pred = model(X)
                val_loss += loss_fn(val_pred, y).item()
                val_correct += (val_pred.argmax(1) == y).type(torch.float).sum().item()

            val_loss /= len(val_dataloader)
            val_correct /= len(val_dataloader.dataset)

            # Final update now belongs to the validation data
            pbar.update(n_batches, values=[("val_loss", val_loss), ("val_acc", val_correct)])
```

#### Train the Model with a Progress Bar Including Validation Data
```python
# Re-instantiate model
model = NeuralNetwork()

# Train the model with progress bar
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.SGD(model.parameters(), lr=1e-3)

epochs = 10
train_loop_pbar_with_val(new_train_dataloader, val_dataloader, model, loss_fn, optimizer, epochs)
```
```
Epoch 1/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 9s 11ms/step - loss: 2.2487 - acc: 0.3137 - val_loss: 2.1909 - val_acc: 0.4718
Epoch 2/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 8s 10ms/step - loss: 2.1157 - acc: 0.5244 - val_loss: 2.0218 - val_acc: 0.5712
Epoch 3/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 8s 10ms/step - loss: 1.8946 - acc: 0.5786 - val_loss: 1.7467 - val_acc: 0.5937
Epoch 4/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 7s 10ms/step - loss: 1.5998 - acc: 0.6092 - val_loss: 1.4567 - val_acc: 0.6229
Epoch 5/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 7s 10ms/step - loss: 1.3470 - acc: 0.6311 - val_loss: 1.2464 - val_acc: 0.6442
Epoch 6/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 8s 10ms/step - loss: 1.1730 - acc: 0.6466 - val_loss: 1.1054 - val_acc: 0.6586
Epoch 7/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 8s 11ms/step - loss: 1.0547 - acc: 0.6574 - val_loss: 1.0077 - val_acc: 0.6715
Epoch 8/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 8s 11ms/step - loss: 0.9714 - acc: 0.6674 - val_loss: 0.9376 - val_acc: 0.6828
Epoch 9/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 8s 11ms/step - loss: 0.9106 - acc: 0.6786 - val_loss: 0.8853 - val_acc: 0.6913
Epoch 10/10
750/750 ━━━━━━━━━━━━━━━━━━━━ 8s 10ms/step - loss: 0.8644 - acc: 0.6892 - val_loss: 0.8449 - val_acc: 0.7011
```

As a sniff test, run the original ``test_loop()`` with the validation dataset and
verify the output is the same as the last epoch in the ``train_loop_pbar_with_val()`` loop:

```python
test_loop(val_dataloader, model, loss_fn)
```
```
Test Error:
 Accuracy: 70.1%, Avg loss: 0.844919
```

### Conclusion
Writing training loops and test loops can be verbose, especially when gathering important metrics to report.
Utilizing Keras's progress bar not only makes for a more visually appealing report, but can
also simplify the amount of code needed for gathering and reporting metrics when training a model.

This recipe tutorial explored how to replace the traditional manual reporting in a
training and test loop with an approach that uses Keras's [``ProgBar`` class](https://keras.io/api/utils/python_utils/#progbar-class).
It was shown how to employ a progress bar with not just a training and testing set,
but also with a validation set to be used during model training.
