---
author: "mventurelli"
layout: "post"
title:  "The curse of image resizing"
slug:   "The curse of image resizing"
date:    2021-08-05 8:00:00
categories: machine-learning
image: images/3dbonet/3d-bonet-architecture.png
tags: machine-learning
---

**Image resizing** is a very common geometrical transformation of an image and it simply consists in a scaling operation.

Since it is one of the most common image processing operations, you can find its implementation in all libraries and considering it is a standard  you can expect that the behavior is well defined and will be the same among the libraries.
Unfortunately, this is not true because there are some little implementation details that differ from library to library. If you are not aware of it, this could create a lot of troubles to your applications.

A tricky scenario that could happens, and we as ML team experimented it, could come from the pre-process step of a machine learning model.
The input of an ML model is usually resized and the main reason is that models train faster on smaller images. An input image that is twice as large requires our network to learn from four times as many pixels, with a lot of more memory need and times that adds up.
Moreover, many deep learning model architectures require that the input must have the same size and raw collected images may usually vary in size.

The workflow of the development an ML model, start from a training phase, for instance in python, then if you satisfy your requirements on the test set, you want to deploy your algorithm.
If you need to use your model in an high-performance environment such as C++, e.g. you need to integrate your model in an existing C++ application, you need a way to export something that could because in the other language. A good idea to be sure to preserve the behavior, is to export all the pipeline, thus not only the forward pass of the network, given by the weights and the architecture of the layers, but also the pre and post processing step.

Fortunately, the main frameworks, i.e. **_Tensorflow_** and **_PyTorch_**, give you the possibility to export all the execution graph into a "program", called `SavedModel` or `TorchScript`, these formats include trained parameters but also the computation.

If you are developing a new model from scratch, you can designed your application in order to export all the pipeline, but if you are using a third party library this is not always possible. So for example, you can export only the inference but not the pre-processing. 

Here comes the problems with the resizing because probably you need to use a different library to resize your input, maybe because you don't know how it is done or there isn't the implementation of the python library in your deploying language.  

## But why the behavior of resize is different?

The differences between implementation comes from how it is done the interpolation.

Typically, to avoid the creation of sampling artifacts, image transformations are done in reverse order that is from destination to source. 
In practice, for each pixel $(x,y)$ of the destination image, you need to compute the coordinates of the corresponding pixel in the input image and copy the pixel value:

$$ I_{dst}(x,y) = I_{src}\left(f_x(x,y), f_y(x,y)\right) $$

where $\langle f_x,f_y \rangle : dst \to src$  is the inverse mapping.
This allow to avoid to have output pixel not assigned to a value.

Usually, when you compute source coordinates you get floating-point numbers, so you need to decide how to choose which source pixel to copy into the destination.

The naive approach, is to round the coordinates to the nearest integers (_nearest-neighbor_ interpolation). However, better results can be achieved by using more sophisticated interpolation methods, where a polynomial function is fit into some neighborhood of the computed pixel $\left(f_x(x,y), f_y(x,y)\right)$, and then the value of the polynomial at $\left(f_x(x,y), f_y(x,y)\right)$ is taken as the interpolated pixel value.

Here come the main problems, different libraries could have some little differences in how they implemented the interpolation filters but above all, if they introduce the anti-aliasing filter. In fact, if we interpreted the image scaling as a form of image resampling from the view of the [Nyquist sampling theorem](https://en.wikipedia.org/wiki/Nyquist%E2%80%93Shannon_sampling_theorem), downsampling to a smaller image from a higher-resolution original can only be carried out after applying a suitable 2D anti-aliasing filter to prevent aliasing artifacts.

`OpenCV` that could be considered a standard de-facto in image processing does not use anti-aliasing filter, instead `Pillow`, probably the most known and used image processing in python, introduce the anti-aliasing filter.



<img src="images/chessboard.png" alt="drawing" width="200"/>




---

[1] On Buggy Resizing Libraries and Surprising Subtleties in FID Calculation (https://arxiv.org/abs/2104.11222)

[2] Geometric Image Transformations https://docs.opencv.org/4.5.2/da/d54/group__imgproc__transform.html

