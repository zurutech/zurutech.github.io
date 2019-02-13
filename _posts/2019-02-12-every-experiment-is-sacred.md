---
author: "manughelfi"
layout: post
did: "blog6"
title:  "Every Experiment is Sacred"
slug:  Every Experiment is sacred
date:   2019-02-12 8:00:00
categories: machine-learning
img: sacred3.jpg
banner: sacred3.jpg
tags: machine-learning
description: "How to manage your Machine Learning experiments"
---

Managing Machine Learning experiments is usually painful.

The usual workflow when approaching a problem using Machine Learning tools is the following. You study the problem, define a possible solution, then you implement that solution and measure its quality. Often the solution depends on some parameters, we will refer to the set of parameters with the term **configuration**. Parameters can include model type, optimizers, learning rate, batch size, steps, losses, and many others.

The configuration influences highly the performance of your solution. If you have enough time and computational power, it is possible to use a Bayesian approach for solving the hyper-parameter selection problem, but in real situations, it is common to perform a limited set of experiment and select the best configuration among them.

In this article we describe how to manage experiments in a good way, ensuring inspectability and reproducibility. We will focus on Machine Learning experiments in python using [Tensorflow](www.tensorflow.com) even if this approach can be applied to all computational experiments.

## Simple (Bad) Solution

To keep track of the configuration the usual way to go (at least in Tensorflow) is to save your models and logs inside a folder with the parameters name getting something like this:
<div markdown="1" class="blog-image-container">
![Sacred](/images/sacred2.png){:class="blog-image"}
</div>
Using this setting with [Tensorboard](https://www.tensorflow.org/guide/summaries_and_tensorboard) (the official Tensorflow visualization tool) it is possible to relate the plot to its configuration. However, this is **not** the right way to manage experiments since it is very hard to see how parameters affect performance. In fact, Tensorboard offers none way to order experiments by configuration, selecting only some experiments or order experiments by performance.
In this way, we do not have any control and tracking of our source code, except for our VCS. Unfortunately, when doing experiments we do not always push our changes since they might be tiny changes or only trials. This can cause issues since our experiments might not be reproducible.

## Sacred Solution

The sacred slogan is:

| *Every experiment is sacred*
| *Every experiment is great*
| *If an experiment is wasted*
| *God gets quite irate*


[Sacred](https://sacred.readthedocs.io/en/latest/index.html) is a tool that lets you configure, organize, log computational experiments. It is designed to introduce a minimal overhead and code addition.

With sacred you can:

- Keep track of all parameters of your experiment
- Run your experiment with different settings
- Save configuration and results in files or database
- Reproduce your results.

Sacred dumps and saves everything into MongoDB including:

- Metrics (training/validation loss, accuracy, or anything else you decide to log)
- Console output
- All the source code you executed
- All the imported library with their versions used at runtime
- All your configuration parameters
- Hardware spec of your host
- Random seeds
- Artifacts and resources.

To visualize and interact with your experiments a nice visualization board for this tool is [Omniboard](https://github.com/vivekratnavel/omniboard).

<div markdown="1" class="blog-image-container">
![Omniboard](/images/omniboard2.png){:class="blog-image"}
</div>

Omniboard lets you tag, annotate and order your experiments making inspection and model selection easy.

## Sacred Integration and experiment design

Integrating sacred in your code is painless and extremely easy. After having installed sacred from pip in your virtual environment you need only to add these lines to your main file:

```python
# other imports

# sacred
from sacred import Experiment
from sacred.observers import MongoObserver

# project imports

# experiment creation and configuration
ex = Experiment("my_experiment")

ex.observers.append(MongoObserver.create())
ex.add_config("default.json")
ex.add_config("experiment.json")


@ex.automain
def main(param1, param2, ...)
    # your code
```

In this way we create a new instance of `Experiment` and an instance of `MongoObserver` in order to store data to MongoDB. We add two different configurations, a default configuration and an experiment configuration. The parameters of the main function are the same parameters described in the `.json` files and are automatically injected by Sacred.