---
author: "mventurelli"
layout: "post"
title:  "The dangers behind image resizing"
slug:   "The dangers behind image resizing"
date:    2021-08-09 8:00:00
categories: machine-learning
image: /images/image-resizing/circle_comparison.png
tags: machine-learning
---

**Image resizing** is a very common geometrical transformation of an image and it simply consists in a scaling operation.

Since it is one of the most common image processing operations, you can find its implementation in all image processing libraries and, considering it is so common you can expect that the behavior is well defined and will be the same among the libraries.  
Unfortunately, this is not true because there are some little implementation details that differ from library to library. If you are not aware of it, this could create a lot of troubles to your applications.

A tricky scenario that could happens, and we as ML team experimented it, could come from the pre-process step of a machine learning model.  
The input of a machine learning model is usually resized mainly because models train faster on smaller images. An input image that is twice as large requires our network to learn from four times as many pixels, with a lot of more memory need and times that adds up.  
Moreover, many deep learning model architectures require that the input must have the same size and it could happen that raw collected images have different sizes.

The workflow of the development an ML model starts from a training phase typically in python then, if you satisfy your requirements on the test set, you want to deploy your algorithm.  
If you need to use your model in an high-performance environment such as C++, e.g. you need to integrate your model in an existing C++ application, or in general you want to use your solution in another programming language[^3], you need a way to export "something" that could be used in the production environment.  
A good idea to be sure to preserve the algorithm behavior is to export all the pipeline, thus not only the forward pass of the network, given by the weights and the architecture of the layers, but also the pre and post processing steps.

Fortunately, the main deep learning frameworks, i.e. **Tensorflow** and **PyTorch**, give you the possibility to export all the execution graph into a "program", called `SavedModel` or `TorchScript`. We use the term program because these formats include not only the architecture and the trained parameters but also the computation.

If you are developing a new model from scratch, you can designed your application in order to export all the pipeline, but if you are using a third party library this is not always possible. So for example, you can export only the inference but not the pre-processing.  
Here comes the problems with the resizing because probably you need to use a different library to resize your input, maybe because you don't know how it is done or there isn't the implementation of the python library in your deploying language.  

## But why the behavior of resize is different?

The definitions of scaling functions are mathematical and should never be a function of the library being used. Unfortunately, implementations differ across commonly-used libraries and mainly comes from how it is done the **interpolation**.

Typically, to avoid the creation of sampling artifacts, image transformations are done in reverse order that is from destination to source.  
In practice, for each pixel $(x,y)$ of the destination image, you need to compute the coordinates of the corresponding pixel in the input image and copy the pixel value:

$$ I_{dst}(x,y) = I_{src}\left(f_x(x,y), f_y(x,y)\right) $$

where $\langle f_x,f_y \rangle : dst \to src$  is the inverse mapping.  
This allow to avoid to have output pixel not assigned to a value.

Usually, when you compute source coordinates you get **floating-point** numbers, so you need to decide how to choose which source pixel to copy into the destination.  
The naive approach, is to round the coordinates to the nearest integers (_nearest-neighbor_ interpolation). However, better results can be achieved by using more sophisticated interpolation methods, where a polynomial function is fit into some neighborhood of the computed pixel $\left(f_x(x,y), f_y(x,y)\right)$, and then the value of the polynomial at $\left(f_x(x,y), f_y(x,y)\right)$ is taken as the interpolated pixel value.

The problem is that different libraries could have some little differences in how they implement the interpolation filters but above all, if they introduce the **anti-aliasing filter**. In fact, if we interpret the image scaling as a form of image resampling from the view of the [Nyquist sampling theorem](https://en.wikipedia.org/wiki/Nyquist%E2%80%93Shannon_sampling_theorem), downsampling to a smaller image from a higher-resolution original can only be carried out after applying a suitable 2D anti-aliasing filter to prevent aliasing artifacts.

`OpenCV` that could be considered the standard de-facto in image processing does not use anti-aliasing filter, instead `Pillow`, probably the most known and used image processing library in python, introduce the anti-aliasing filter.


## Comparison of libraries

To have an idea on how the different implementations affect the resized output image, we made a comparison of four libraries, the ones we considered the most used, in particular in the ML fields. Moreover, we focused on libraries that could be used from python.

We tested the following libraries and methods:
1. **OpenCV** v4.5.3: [`cv2.resize`](https://docs.opencv.org/4.5.2/da/d54/group__imgproc__transform.html#ga47a974309e9102f5f08231edc7e7529d)
2. **Pillow Image Library** (PIL) v8.2.0: [`Image.resize`](https://pillow.readthedocs.io/en/stable/reference/Image.html#PIL.Image.Image.resize)
3. **TensorFlow** v2.5.0: [`tf.image.resize`](https://www.tensorflow.org/api_docs/python/tf/image/resize)

    This method has a flag `antialias` to enable the anti-aliasing filter (default is false), we tested the method in both cases, either flag disabled and enabled.

4. **PyTorch** v1.9.0: [`torch.nn.functional.interpolate`](https://pytorch.org/docs/stable/generated/torch.nn.functional.interpolate.html)

    PyTorch has another method for resizing, that is `torchvision.transforms.Resize`, we decided do not use for the tests because is a wrapper around the PIL library, so the results will the same to `Pillow`.


Each method supports some filters, so we choose a common subset and we tested the libraries with them. In particular, we choose:
- **_nearest_**: the most naive approach that often leads to aliasing. The high-frequency information in the original image becomes misrepresented in the downsampled image.
- **_bilinear_**: linear filter that works by interpolating pixel values, introducing a continuous transition into the output even where the original image has discrete transitions. It is an extension of linear interpolation to a rectangular grid.
- **_bicubic_**: similar to _bilinear_ filter but in contrast to it, which only takes 4 pixels (2×2) into account, bicubic interpolation considers 16 pixels (4×4). Images resampled with bicubic interpolation should be smoother and have fewer interpolation artifacts.
- **_lanczos_**: calculate the output pixel value using a high-quality _Lanczos_ filter (a truncated sinc) on all pixels that may contribute to the output value, typically it is used a 8x8 neighborhood.
- **_box_** (aka **_area_**): is the simplest linear filter; while better than naive subsampling above, it is typically not used. Each pixel of source image contributes to one pixel of the destination image with identical weights.

### Qualitative results

To better see the different results obtained with the presented functions, we follow the test made in [^1]. We create a synthetic image 128x128 with a circle of thickness 1 pixel and we resized with a downscaling factor of 4 (destination image will be 32x32).  
We decided to focus on downsampling because it involves throwing away information and is more error-prone.  
Here you can see the qualitative results of our investigation:


<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="comparison on circle" src="/images/image-resizing/circle_comparison.png" width="100%">
        <figcaption>Comparison of methods on synthetic image of a circle.</figcaption>
    </figure>
</div>

In the picture we add an icon to highlight if the downsampling method antialias and remove high-frequencies in the correct way. We use a green icon to show good results, yellow if the filter does not provide sufficient antialiasing, red if artifacts are introduced in the low-resolution image.  

As you can see, even downsampling a simple image of a circle provides wildly inconsistent results across different libraries.  
It seems that `Pillow` is the only one that performs downsampling in the right way in fact, apart for the _nearest_ filter that for definition does not apply antialiasing, all the others does not introduce artifacts and produces a connected ring.

`OpenCV` and `PyTorch` achieve very similar results for all the filters and both does not use antialiasing filter.  
This sparse input image shows that these implementations do not properly blur, resulting in a disconnected set of dots after downsampling given a poor representation of the input image.

`TensorFlow` has an hybrid behavior, if you don't enable the antialiasing flag (first row) the results are the same of `OpenCV`, instead if you enable it you get the same results of `Pillow`.

I find interesting that there are differences even for a filter like the _nearest neighbor_ that is quite straightforward to implement. Whilst the box filter seems to give the same results among the libraries.  
We can consider the methods as two categories, the ones that have Pillow-like results and the ones similar to `OpenCV`.

We further evaluate the libraries with other synthetic images with different patterns.  
The first one is a grid of white square 50x50 with 5 pixel of black border.

<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="comparison on grid" src="/images/image-resizing/grid_comparison.png" >
        <figcaption>Comparison of methods on synthetic image of a grid.</figcaption>
    </figure>
</div>

The previous consideration are valid also in this case, and it clear how without the antialiasing the resulting images do not represent correctly the input.
If you look at `OpenCV` results, it is curious that are in practice the same results given by the _nearest neighbor_ filter of `Pillow`, it's like incorrectly implemented filters inadvertently resemble the naive approach.

Other interesting results could be achieve using a pattern similar to the previous grid but with a border enlarged up to 22 pixels and with 20x20 white squares (note that we use a slightly different value for the border and the squares).

<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="comparison on grid" src="/images/image-resizing/grid2_comparison.png" >
        <figcaption>
        Comparison of Pillow and OpenCV on a grid image.
        From left to right, raw image (652x652), Pillow resized image with nearest filter, OpenCV resized image with nearest filter, Pillow resized image with blinear filter and OpenCV resized image with bilinear filter.<br>
        All the low-resolution images are 64x64 and only for a better view, are displayed with the same size of raw image.
        </figcaption>
    </figure>
</div>

In this case, in the resulting images the number of square is unchanged but in some cases some rows or columns result having lower size.

### Natural images

We choose to use synthetic images because we want to enhance the creation of artifacts and see the differences between the libraries. These differences are more difficult to see on natural images, i.e. the ones acquired with a camera. In this case, as you see can in the figures below, the results with the _nearest neighbor_ filter are quite the same instead, using the _bilinear_ the effect of the antialasing filter become more visible.

<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="nearest neighbor on dog" src="/images/image-resizing/dog_nearest.png" >
        <figcaption>Nearest neighbor interpolation on natural image.</figcaption>
    </figure>
</div>
<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="bilinear on dog" src="/images/image-resizing/dog_bilinear.png" >
        <figcaption>Bilinear interpolation on natural image.</figcaption>
    </figure>
</div>

## Pillow Resize

Since we are interested in deploying our applications in C++, and we noted that the most correct behavior is given by the `Pillow` resize, we would like to use it in C++.  
The `Pillow` [image processing](https://github.com/python-pillow/Pillow/blob/master/src/libImaging/) algorithms are almost all written in C, but they cannot be directly used because they are designed to be part of the python wrapper.  
We therefore released a porting of the resize method in a new standalone library that works on `cv::Mat` so it will be compatible with all `OpenCV` algorithms.

TODO: insert correct link to repo

We want to remark that the right way to use the resize in an ML application is the export the pre-processing step of the algorithm, in this way you are sure that your python model works in the same manner of your deployed model.



# References

[^1]: [On Buggy Resizing Libraries and Surprising Subtleties in FID Calculation](https://arxiv.org/abs/2104.11222)

[^2]: [Geometric Image Transformations](https://docs.opencv.org/4.5.2/da/d54/group__imgproc__transform.html)

[^3]: [Deploy and Train TensorFlow models in Go: Human Activity Recognition case study](https://pgaleone.eu/tensorflow/go/2020/11/27/deploy-train-tesorflow-models-in-go-human-activity-recognition/)



<br><br>
_I want to acknowledge Guido Salto for helping in the choice of the patterns that better highlight the artifacts and the differences in the results._
