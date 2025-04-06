---
layout: post
title: "Shifting-left in ML: catching configuration errors"
date: 2025-04-05 15:09:00
description: 
tags:
  - software-engineering
  - pants
categories: 
featured: false
---
In software engineering, [**shifting left**](https://en.wikipedia.org/wiki/Shift-left_testing) means moving error detection *earlier* in the development lifecycle—closer to where the problems originate. Instead of discovering bugs in production (the far right of the timeline), you catch them during development or testing.

This mindset is especially valuable in **machine learning**, where a single configuration mistake—like a typo or invalid file path—might not cause an error until **after hours of training**. At that point, you've burned time and compute. You have to fix things and re-launch the experiment.

One powerful, low-overhead way to shift-left is to **validate your configurations early**, before you kick off training. I'm going to show you how I do it simply with the `attrs` library.

---

## What Can Go Wrong?

Consider a typical ML training script using Hydra for configuration:

```python
@hydra.main(config_path=".", config_name="config", version_base=None)
def main(cfg: DictConfig):
    model = MyModel()
    train_dataset = MyDataset(path=cfg.train_dataset_path)
    train(model, train_dataset)

    test_dataset = MyDataset(cfg.test_dataset_path)
    test_metric = F1Score(average=cfg.f1_average_kind)
    test(model, test_dataset)
```

And the relevant configuration file:
```yaml
# config.yaml
train_data_path: data/train.csv
test_data_path: data/test.csv
f1_average_kind: micro  # can be either micro or macro
```

This works fine *until*:

- Someone sets `f1_average_kind: "micor"` (a typo),
- Or `test_dataset_path: "data/missing.csv"` (a file that doesn’t exist).

These mistakes **don’t cause immediate failures**. Training proceeds happily, and the error might not appear until testing—hours later.

---

## A Quick Primer on `attrs` and Validators

[`attrs`](https://www.attrs.org/) is a Python library that helps you write cleaner, more declarative class definitions. It’s similar to `dataclasses`, but more flexible and extensible. One of its standout features is **[validators](https://www.attrs.org/en/stable/examples.html#validators)**.

Validators in `attrs` are functions that run automatically when an object is initialized. They let you enforce constraints like:

- Ensuring a field has the correct type
- Making sure a value is within a valid range
- Checking for existence of a file path
- Any other custom logic you need

Here’s a basic example:

```python
from attrs import define, field, validators

@define
class Person:
    name: str = field()
    age: int = field()

    @name.validator:
    def name_is_str(self, attribute, value):
        if not isinstance(value, str):
            raise TypeError("name is not of type string.")

    @age.validator
    def valid_age(self, attribute, value):
        if not isinstance(value, int):
            raise TypeError("age is not of type int.")
        if value < 0:
            raise ValueError("age must be greater than or equal to zero.")
```

When you try to create a `Person` with invalid data:

```python
Person(name=123, age=-5)
```

You'll get a helpful error right away. This is exactly the kind of behavior we want when dealing with ML configurations.

`attrs` also offers a lot of useful built in validators, so the above logic can be written in the following way:

```python
@define
class Person:
    name: str = field(validator=validators.instance_of(str))
    age: int = field(validator=[validators.instance_of(int), validators.ge(0)])
```

For this article, we'll just use the `@attribute.validator` decorator.

---

## How to Shift Left with `attrs` Validators

We can apply the shifting-left concept to ML experiment configs. By defining your configuration as an `attrs` class, you can use validators to enforce correctness up front.

```python
from attrs import define, field
from pathlib import Path
from hydra.utils import instantiate


@define(auto_attribs=True)
class ExperimentConfig:
    train_data_path: Path = field()
    test_data_path: Path = field()
    f1_average_kind: str = field()

    @train_data_path.validator
    def train_data_exists(self, attribute, value):
        assert value.exists(), f"Training data at {value} doesn't exist."

    @test_data_path.validator
    def test_data_exists(self, attribute, value):
        assert value.exists(), f"Test data at {value} doesn't exist."

    @f1_average_kind.validator
    def is_average_valid(self, attribute, value):
        assert value in ("micro", "macro"), f"Invalid f1_average_kind: {value}"
```

Now, we don't want to change our basic set-up too much. We still want to be able to use hydra to configure our experiments from the command line. Thankfully, we don't need to change things too much. 

We can use hydra's `instantiate` function with `_target_` inside the yaml configuration to instantiate any Python object ([reference](https://hydra.cc/docs/advanced/instantiate_objects/overview/)).

In our case, we add the `_target_` field.

```yaml
# config.yaml
_target_: ExperimentConfig  # add _target_ field

train_data_path: data/train.csv
test_data_path: data/missing.csv  # <- file doesn't exist
f1_average_kind: micor            # <- typo
```

Then running `instantiate` returns an `ExperimentConfig` object.

```python
from hydra.utils import instantiate 

@hydra.main(config_path=".", config_name="config", version_base=None)
def main(cfg: DictConfig):
    # returns an ExperimentConfig object
    cfg: ExperimentConfig = instantiate(cfg)
```

The reason this is so important in our case is that when hydra instantiates the `ExperimentConfig` object, it also triggers all the `attrs` validators that we added to the class. Now, the script fails fast with a clear error before doing any expensive work.

This is what our script looks like with all of the changes:

```python
from hydra.utils import instantiate 

@hydra.main(config_path=".", config_name="config", version_base=None)
def main(cfg: DictConfig):
    # triggers validators because we instantiate the ExperimentConfig object
    cfg: ExperimentConfig = instantiate(cfg)

    model = MyModel()
    train_dataset = MyDataset(cfg.train_data_path)
    train(model, train_dataset)

    test_dataset = MyDataset(cfg.test_data_path)
    test_metric = F1Score(average=cfg.f1_average_kind)
    test(model, test_dataset)
```
---
## Final Thoughts

When you're working on machine learning projects, your time and compute budget are some of your most valuable resources. Simple configuration mistakes—like missing files or invalid parameters—can silently waste hours of work.

By shifting left and validating your configurations early, you catch these issues before they turn into real problems. With tools like `attrs`, it's easy to build safeguards directly into your config logic. This means fewer reruns, faster feedback loops, and more time spent on meaningful work—not debugging.
