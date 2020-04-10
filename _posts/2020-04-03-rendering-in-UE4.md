---
author: "alepaoletti"
layout: post
did: "blog10"
title: "Rendering in Unreal Engine 4"
slug: "RenderingInUE4"
date: 2020-04-06 08:00:00
categories: graphics
img: banners/GPU_profiler.png
banner: banners/GPU_profiler.png
tags: graphics
description: "A quick overview of the rendering pipeline in Unreal Engine 4."
---

**Unreal Engine 4** contains a really cool multi-platform and multi-threaded rendering engine.

The code for rendering exists in the *Renderer* module, which is compiled to a shared library (e.g. *.dll* on Windows) to allow faster iteration. The *Renderer* module depends on the *Engine* module because it has many callbacks into it. However, when the *Engine* needs to call some code in the *Renderer*, this happens through the *IRendererModule* interface.

The *RHI (Render Hardware Interface)* is the other key module for graphics programming as it is the interface for rendering APIs. Here is where the magic happens to allow the abstract rendering code to work on different platforms using different APIs.

Rendering in Unreal it's a complex area, especially if you want to make modifications to customize it for your own purposes. You have to take care of every memory read and write not only to ensure thread safety, but also to avoids race conditions. Here is the complete UE4 documentation for graphics programming: [UE4 Graphics Programming](https://docs.unrealengine.com/en-US/Programming/Rendering/index.html).

The renderer code runs in a separate thread, the *Rendering Thread*. It operates in parallel with the game thread and it's usually one or two frames behind it. It serves to enqueue platform-agnostic render commands into the renderer's command list through the `ENQUEUE_RENDER_COMMAND` macro. To make sure your code is called by the right thread, you can add checks such as `check(IsInGameThread())` or `check(IsInRenderingThread())` for improving code stability.

Finally, a new thread, the *RHI Thread*, executes these commands via the appropriate graphics API on the backend.

To know more about this, check the Unreal documentation: [Parallel Rendering](https://docs.unrealengine.com/en-US/Programming/Rendering/ParallelRendering/index.html).

### Why am I writing this

There's not a lot of documentation around the UE4 renderer module, especially because it's constantly under development. Most of the code is private, and it changes every release.
For instance, in Unreal Engine 4.22 the mesh drawing pipeline changed completely: this [GDC video](https://www.youtube.com/watch?v=qx1c190aGhs&feature=youtu.be) shows the refactor that happened (and here is the official documentation [Mesh Drawing Pipeline](https://docs.unrealengine.com/en-US/Programming/Rendering/MeshDrawingPipeline/index.html)).

In this blog, I would like to keep track of the changes and the updates, and share some tips or gotchas that I find useful while developing. Writing stuff down will help me remember, and hopefully it will be useful for someone else.

## Rendering Flow

As briefly described above, Unreal uses three main threads to renders a frame.

TODO: Image here with a brief description

The render of a frame starts in the main game thread:

**Game thread (CPU)**: it calculates all the transformations of the objects in the scene. This includes animations, physics, AI, and all the logic that happens in our application.

**Draw thread (CPU)**: it calculates what to include in the rendering. This is mostly done on the CPU, but there are some parts on the GPU as well. The occlusion process happens in this thread and it happens *per object*. A list of all visible objects is built after 4 occlusion steps:

1. *Distance Culling* - Removes objects further than a given distance from the camera
2. *Frustum Culling* - Removes objects outside the camera frustum
3. *Precomputed Visibility* - Divides the scene into a grid of cells and every cell stores what it could be visible from that position
4. *Occlusion Culling* - Check the visibility state of every object, if it's occluded by another one at that moment

To debug these occlusion steps, you can use `stat initviews` that gives you the expense and the counts of occluded objects of this render step.

Now that the engine knows what objects needs to be rendered and where they are, it remains to know in what order the renderer need to render those: this is done as first step in the next section.

**Render thread (CPU - GPU)**: the geometry rendering starts. Unreal have both *forward* and *deferred* rendering paths.

TODO: explain briefly the difference and why matters here (number of passes)

1. *Prepass/Early Z Pass*: GPU figure out which models will be displayed where. It's done by a pass of z-depth, to avoid pixel overdraw.

2. *Draw Calls*: GPU starts to render object by object, firing a drawcall. This consist of a group of polygons that shares the same properties. Between every drawcall, the render state could change (meaning that the render needs to set the different properties for the actual drawcall). Behind the scene, Unreal sorts the drawcalls to optimize the number of state changes. The fewer changes the engine needs to do, the better are the performances. Usually, this optimization consist in grouping drawcalls by materials, and the engine does this automatically.

    Drawcalls have a huge impact on performance: every time the render is done executing the commands, it needs to receive the new commands for the next call from the render thread and this adds a significant overhead.

    With `stat RHI` you can check the number of drawcalls that are called from the renderer. Broadly (because it depends on a lot of different variables) a reasonable number is 2000/3000 for a desktop application. 5000 is still acceptable for good hardware; over 10000 it's probably a problem. On mobile, a good number is far lower, about a few hundred.

    Another very useful tool is [RenderDoc](https://renderdoc.org/): it's a free standalone graphics debugger that allows quick and easy single-frame capture of a single frame to make a detailed inspection of all the calls the application does; this works with every render API. Unreal has a plugin for it, see more [here](https://docs.unrealengine.com/en-US/Engine/Performance/RenderDoc/index.html).

This is a very big topic, so for drawcalls I would finish here.

The thing to pay attention is that in real-time rendering objects are rendered one by one, and for the Unreal Engine renderer an object is a primitive component, not an actor. This means that when you have a blueprint with 5 static mesh components in it with a single material each, that will results in 5 draw calls (at least, it's actually 5 drawcalls for every pass the object is rendered into). Plus, if you have a single static mesh component with two materials assigned, then you'll have 2 drawcalls, one per material.

From version 4.22, Unreal developed an automatic draw call merging system, but this works with some limitations (more [here](https://dq8iqaixvew1d.cloudfront.net/en-US/Programming/Rendering/MeshDrawingPipeline/index.html)).

## Rendering Elements

### Textures

It's important to understand what a texture actually is for the renderer, because it influences a lot the memory consumption of the application and the visual quality of the final image.

When rendering, we have limited memory and bandwidth: it's essential to compress an image before using it on the GPU. Unreal does this automatically during the file import. A texture is compressed with a compression algorithm that differ per platform: the most common used on PC are *DXTC*/*BC* (*DirectX Texture Compression*/*Block Compression*). In very simple words, these algorithms act by compressing blocks of pixels into smaller ones to store less informations with a minimum loss percentage. Without digging more in technical details, what you need to know is that you can change the texture compression by opening a texture asset from the content browser.

For every texture, the engine automatically creates **mipmaps (or MIP maps)**. "MIP" stands for the Latin phrase *multum in parvo*, meaning "much in little", referring to the fact that mipmaps solve the texture minification/magnification problem, that creates a really annoying aliasing effect.
These maps are generated by creating a smaller image with height and width a power of two smaller than the original one: every level is a quarter smaller in size and represent a mipmap level. To do this efficiently, textures should be power of two (1024x1024, 512x512, 256x256 and so on). Please note that is not necessary that the image is squared, it could be rectangular as well: the important thing is that the sizes must be a power of two (for instance, a texture could be 256x1024, and that's still good for mipmaps generation).

### Shaders and Materials

### Lighting

### Post Processing Effects
