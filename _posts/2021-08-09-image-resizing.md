---
author: "mventurelli"
layout: "post"
title:  "The dangers behind image resizing"
slug:   "The dangers behind image resizing"
date:    2021-08-09 8:00:00
categories: machine-learning
image: images/image-resizing/circle_comparison.png
tags: machine-learning
---

**Image resizing** is a very common geometrical transformation of an image, and it simply consists of a scaling operation.

Since it is one of the most common image processing operations, you can find its implementation in all image processing libraries. Because it is so common, you can expect that the behavior is well defined and will be the same among the libraries.  
Unfortunately, this is not true because some little implementation details differ from library to library. If you are not aware of it, this could create a lot of trouble for your applications.

A tricky scenario that could happen, and we as the ML team experienced it, could come from the pre-processing step of a machine learning model.  
Usually, we resize the input of a machine learning model mainly because models train faster on smaller images. An input image that is twice the size requires our network to learn from four times as many pixels, with more memory need and times that add up. 
Moreover, many deep learning model architectures require that the input have the same size, and raw collected images might have different sizes.

The workflow of the development of an ML model starts from a training phase, typically in Python. Then, if your metrics on the test set satisfy your requirements, you may want to deploy your algorithm.  
Suppose you need to use your model in a high-performance environment such as C++, e.g., you need to integrate your model in an existing C++ application or in general. In that case, you want to use your solution in another programming language [^3], and you need a way to export "something" that could be used in the production environment.  
A good idea to preserve the algorithm behavior is to export the whole pipeline, thus not only the forward pass of the network, given by the weights and the architecture of the layers but also the pre-and post-processing steps.

Fortunately, the main deep learning frameworks, i.e., **Tensorflow** and **PyTorch**, give you the possibility to export the whole execution graph into a "program," called `SavedModel` or `TorchScript`, respectively. We used the term program because these formats include both the architecture, trained parameters, and computation.

If you are developing a new model from scratch, you can design your application to export the entire pipeline, but this is not always possible if you are using a third-party library. So, for example, you can export only the inference but not the pre-processing.
Here come the resizing problems because you probably need to use a different library to resize your input, maybe because you don't know how the rescaling is done, or there isn't the implementation of the Python library in your deploying language.  

## But why the behavior of resizing is different?

The definition of scaling function is mathematical and should never be a function of the library being used. Unfortunately, implementations differ across commonly-used libraries and mainly come from how it is done the **interpolation**.

Image transformations are typically done in reverse order (from destination to source) to avoid sampling artifacts. 
In practice, for each pixel $$(x,y)$$ of the destination image, you need to compute the coordinates of the corresponding pixel in the input image and copy the pixel value:

$$ I_{dst}(x,y) = I_{src}\left(f_x(x,y), f_y(x,y)\right) $$

where $$\langle f_x,f_y \rangle : dst \to src$$ is the inverse mapping.  
This allows avoiding to have output pixels not assigned to a value.

Usually, when you compute source coordinates, you get **floating-point** numbers, so you need to decide how to choose which source pixel to copy into the destination.  
The naive approach is to round the coordinates to the nearest integers (_nearest-neighbor_ interpolation). However, better results can be achieved by using more sophisticated interpolation methods, where a polynomial function is fit into some neighborhood of the computed pixel $$\left(f_x(x,y), f_y(x,y)\right)$$, and then the value of the polynomial at $$\left(f_x(x,y), f_y(x,y)\right)$$ is taken as the interpolated pixel value [^2].

The problem is that different library could have some little differences in how they implement the interpolation filters but above all, if they introduce the **anti-aliasing filter**. In fact, if we interpret the image scaling as a form of image resampling from the view of the [Nyquist sampling theorem](https://en.wikipedia.org/wiki/Nyquist%E2%80%93Shannon_sampling_theorem), downsampling to a smaller image from a higher-resolution original can only be carried out after applying a suitable 2D anti-aliasing filter to prevent [aliasing artifacts](https://en.wikipedia.org/wiki/Aliasing).

`OpenCV` that could be considered the standard de-facto in image processing does not use an anti-aliasing filter. On the contrary, `Pillow`, probably the most known and used image processing library in Python, introduces the anti-aliasing filter.


## Comparison of libraries

To have an idea of how the different implementations affect the resized output image, we compared four libraries, the ones we considered the most used, in particular in the ML field. Moreover, we focused on libraries that could be used in Python.

We tested the following libraries and methods:
1. **OpenCV** v4.5.3: [`cv2.resize`](https://docs.opencv.org/4.5.2/da/d54/group__imgproc__transform.html#ga47a974309e9102f5f08231edc7e7529d)
2. **Pillow Image Library** (PIL) v8.2.0: [`Image.resize`](https://pillow.readthedocs.io/en/stable/reference/Image.html#PIL.Image.Image.resize)
3. **TensorFlow** v2.5.0: [`tf.image.resize`](https://www.tensorflow.org/api_docs/python/tf/image/resize)

    This method has a flag `anti-alias` to enable the anti-aliasing filter (default is false). We tested the method in both cases, either flag disabled and enabled.

4. **PyTorch** v1.9.0: [`torch.nn.functional.interpolate`](https://pytorch.org/docs/stable/generated/torch.nn.functional.interpolate.html)

    PyTorch has another method for resizing, that is `torchvision.transforms.Resize`. We decided not to use it for the tests because it is a wrapper around the PIL library, so that the results will be the same compared to `Pillow`.


Each method supports a set of filters. Therefore, we chose a common subset present in almost all libraries and tested the functions with them. In particular, we chose:
- [**_nearest_**](https://en.wikipedia.org/wiki/Nearest-neighbor_interpolation): the most naive approach that often leads to aliasing. The high-frequency information in the original image becomes misrepresented in the downsampled image.
- [**_bilinear_**](https://en.wikipedia.org/wiki/Bilinear_interpolation): a linear filter that works by interpolating pixel values, introducing a continuous transition into the output even where the original image has discrete transitions. It is an extension of linear interpolation to a rectangular grid.
- [**_bicubic_**](https://en.wikipedia.org/wiki/Bicubic_interpolation): similar to _bilinear_ filter but in contrast to it, which only takes 4 pixels (2×2) into account, bicubic interpolation considers 16 pixels (4×4). Images resampled with bicubic interpolation should be smoother and have fewer interpolation artifacts.
- [**_lanczos_**](https://en.wikipedia.org/wiki/Lanczos_resampling): calculate the output pixel value using a high-quality _Lanczos_ filter (a truncated sinc) on all pixels that may contribute to the output value. Typically it is used on an 8x8 neighborhood.
- [**_box_**](https://en.wikipedia.org/wiki/Image_scaling#Box_sampling) (aka **_area_**): is the simplest linear filter; while better than naive subsampling above, it is typically not used. Each pixel of the source image contributes to one pixel of the destination image with identical weights.

### Qualitative results

To better see the different results obtained with the presented functions, we follow the test made in [^1]. We create a synthetic image 128x128 with a circle of thickness 1 pixel, and we resized it with a downscaling factor of 4 (destination image will be 32x32).  
We decided to focus on downsampling because it involves throwing away information and is more error-prone.  

Here you can see the qualitative results of our investigation:


<div class="blog-image-container">
    <figure id="figure1">
        <img class="blog-image" alt="comparison on circle" src="/images/image-resizing/circle_comparison.png" width="100%">
        <figcaption>Figure 1. Comparison of methods on the synthetic image of a circle.</figcaption>
    </figure>
</div>

In [Figure 1](#figure1), we added an icon to highlight if the downsampling method antialiases and remove high-frequencies in the correct way. We used a green icon to show good results, yellow if the filter does not provide sufficient antialiasing, red if artifacts are introduced in the low-resolution image.  

As you can see, even downsampling a simple image of a circle provides wildly inconsistent results across different libraries.  
It seems that `Pillow` is the only one that performs downsampling in the right way in fact, apart from the _nearest_ filter that for definition does not apply antialiasing, all the others do not introduce artifacts and produce a connected ring.

`OpenCV` and `PyTorch` achieve very similar results for all the filters, and both do not use antialiasing filters.    
This sparse input image shows that these implementations do not properly blur, resulting in a disconnected set of dots after downsampling gave a poor representation of the input image.

`TensorFlow` has a hybrid behavior. If you don't enable the antialiasing flag (first row), the results are the same as `OpenCV`. Instead, if you enable it, you get the same results as `Pillow`.

I find it interesting that there are differences even for a filter like the _nearest neighbor_ that is quite straightforward to implement. At the same time, the box filter seems to give the same results among the libraries.  
Eventually, we can consider the methods as two categories: Pillow-like results and the ones similar to `OpenCV`.

We further evaluate the libraries with other synthetic images with different patterns. 
The first one (Figure 2) is a grid of white square 50x50 with 5 pixels of black border.

<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="comparison on grid" src="/images/image-resizing/grid_comparison.png" >
        <figcaption>Figure 2. Comparison of methods on the synthetic image of a grid.</figcaption>
    </figure>
</div>

The previous considerations are valid also in this case, and it is clear how without the antialiasing the resulting images do not represent correctly the input.
If you look at `OpenCV` results, it is curious that they are, in practice, the same results given by the _nearest neighbor_ filter of `Pillow`. It seems like that incorrectly implemented filters inadvertently resemble the naive approach.

Other interesting results can be achieved using a pattern similar to the previous grid but with a border enlarged to 22 pixels and 20x20 white squares (note that we use a slightly different value for the border and the squares).

<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="comparison on grid" src="/images/image-resizing/grid2_comparison.png" >
        <figcaption>
        Figure 3. Comparison of Pillow and OpenCV on a grid image.
        From left to right, raw image (652x652), Pillow resized image with the nearest filter, OpenCV resized image with the nearest filter, Pillow resized image with the bilinear filter and OpenCV resized image with the bilinear filter.  
        All the low-resolution images are 64x64 and are displayed with the same size as the raw image only for a better view.
        </figcaption>
    </figure>
</div>

With this input, the number of squares in the resulting images is unchanged, but in some cases, some rows or columns have a rectangular shape.

### Natural images

We chose to use synthetic images because we want to evidence the creation of artifacts and see the differences between the libraries. These differences are more difficult to see on natural images, i.e., the ones acquired with a camera. In this case, as you see in the figures below, the results with the _nearest neighbor_ filter are quite the same while, using the _bilinear_ one, the effect of the antialiasing filter becomes more visible.

<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="nearest neighbor on dog" src="/images/image-resizing/dog_nearest.png" >
        <figcaption>Figure 4. Nearest neighbor interpolation on a natural image.</figcaption>
    </figure>
</div>
<div class="blog-image-container">
    <figure>
        <img class="blog-image" alt="bilinear on dog" src="/images/image-resizing/dog_bilinear.png" >
        <figcaption>Figure 5. Bilinear interpolation on a natural image.</figcaption>
    </figure>
</div>

<br/>

## Conclusion

In this post, we saw that even if image resizing is one of the most used image processing operations, it could hide some traps. It is essential to carefully choose the library we want to use to perform resize operations, particularly if you're going to deploy your ML solution.

Since we noticed that the most correct behavior is given by the `Pillow` resize and we are interested in deploying our applications in C++, it could be useful to use it in C++. 
The `Pillow` [image processing](https://github.com/python-pillow/Pillow/blob/master/src/libImaging/) algorithms are almost all written in C, but they cannot be directly used because they are designed to be part of the Python wrapper.  
We, therefore, released a porting of the resize method in a new standalone library that works on `cv::Mat` so it would be compatible with all `OpenCV` algorithms.  
You can find the library [here](TODO: insert correct link to repo).

We want to remark again that the right way to use the resize operation in an ML application is to export the algorithm's pre-processing step. In this way, you are sure that your Python model works similarly to your deployed model.


<br/>

_I want to acknowledge Guido Salto for helping in the choice of the patterns that better highlight the artifacts and the differences in the results._

<br/><br/>

# References

[^1]: [On Buggy Resizing Libraries and Surprising Subtleties in FID Calculation](https://arxiv.org/abs/2104.11222)

[^2]: [Geometric Image Transformations](https://docs.opencv.org/4.5.2/da/d54/group__imgproc__transform.html)

[^3]: [Deploy and Train TensorFlow models in Go: Human Activity Recognition case study](https://pgaleone.eu/tensorflow/go/2020/11/27/deploy-train-tesorflow-models-in-go-human-activity-recognition/)
