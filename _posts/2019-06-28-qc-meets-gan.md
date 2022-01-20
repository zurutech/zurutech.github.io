---
author: "ml-lab"
layout: post
title: "Quality Control meets Generative Intelligence"
slug: Quality Control meets Generative Intelligence
date: 2019-06-28 8:00:00
categories: machine-learning
image: images/gans/anomaly.png
tags: machine-learning
description: "How to use GANs in the anomaly detection domain."
---

At Zuru Tech, our construction project doesn't stop. Our factories continue to change quickly to find the best solutions to the severe problems we face every day. Continuous evolution means that we are always attentive to every little change. Moreover, these changes must not affect the quality of our products. For this reason, we have invested heavily in innovative anomaly detection algorithms that allow us to quickly recognize production processes and maintain the level of quality, at all times, at its best.

The real challenge in recognizing anomalies is that most of the time, you can not predict what kind of problem may arise. And when you do not know what to expect precisely, it is challenging to design software that can assess the quality of a product. After all, how do we instruct a machine to recognize a problem that neither do we know what it could be?

That's why we decided to make the machines as smart as possible. It was a short step. We did a lot of research and using AI techniques, in a mix between Computer Vision and Deep Learning, we realized how one of the most innovative technologies of recent years could be really effective in our case. We are talking about Generative Adversarial Networks (GANs). They are mostly known for algorithms for generating works of art

<div markdown="1" class="blog-image-container">
![gan art](/images/gans/art.png){:class="blog-image"}
</div>

or for the much talked about video fakes

<div markdown="1" class="blog-image-container">
![gan fakes](/images/gans/fakes.png){:class="blog-image"}
</div>

but their use in the search for anomalies is beginning to tease the minds of more than a single researcher. By putting together some work, we were able to get great results where the most classic algorithms of Machine Learning have already been well squeezed.

We can't go into detail about how we decided to use them or where exactly we apply them, but our work has resulted in a survey that wants to relate the most successful work in the field of quality control through the use of GANs. Also, we will provide a toolbox for anyone who wants to try one of the many possible implementations and then choose, case by case, the one with the best results. The toolbox, developed with the new TensorFlow 2.0 and Python, will be soon released.

You can find our work published at the following address: [https://arxiv.org/abs/1906.11632](https://arxiv.org/abs/1906.11632)
