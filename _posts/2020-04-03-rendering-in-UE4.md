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
description: "A really quick overview of the rendering pipeline in Unreal Engine 4."
---

Unreal Engine 4 has a lot of code for doing rendering: it exists in its own module - the *Renderer* module - which is compiled to a shared library (e.g. *.dll* on Windows) to allow faster iteration. The *Renderer* module depends on the Engine module because it has many callbacks into Engine. However, when the Engine needs to call some code in the Renderer, this happens through the *IRendererModule* interface.

The *RHI (Render Hardware Interface)* module is the other key module for graphics programming as it is the interface for rendering APIs. Here is where the magic happens to allow the abstract rendering code to work on different platforms using different APIs.

The renderer code runs in a separate thread, the *Rendering Thread*. It operates in parallel with the game thread and it's usually one or two frames behind it. It serves to enqueue platform-agnostic render commands into the renderer's command list through the `ENQUEUE_RENDER_COMMAND` macro. To make sure your code is called by the right thread, you can add checks such as `check(IsInGameThread())` or `check(IsInRenderingThread())` for improving code stability.

Finally, a new thread, the *RHI Thread*, executes these commands via the appropriate graphics API on the backend.

It's a complex topic, especially if you want to make modifications to customize it for your own purposes.

Here is the official UE4 documentation: [UE4 Graphics Programming](https://docs.unrealengine.com/en-US/Programming/Rendering/index.html).

### Why am I writing this

There's not a lot of documentation around the UE4 renderer module, especially because it's constantly under development. Most of the code is private, and it changes every release.
For instance, in Unreal Engine 4.22 the mesh drawing pipeline changed completely: this [GDC video](https://www.youtube.com/watch?v=qx1c190aGhs&feature=youtu.be) shows the refactor that happened (and here is the official documentation [Mesh Drawing Pipeline](https://docs.unrealengine.com/en-US/Programming/Rendering/MeshDrawingPipeline/index.html)).

In this blog I would like to keep track of the changes and the updates, and share some tips or gotchas that I find useful while developing. Writing stuff down will help me remember, and hopefully it will be useful for someone else.
