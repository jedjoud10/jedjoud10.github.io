+++
title = "How I implemented a compute shader based memory allocator in Unity"
date = 2024-03-01
draft = true

[taxonomies]
categories = ["Unity", "Compute Shaders", ]
tags = ["game-dev", "unity"]

[extra]
lang = "en"
toc = true
comment = true
copy = true
math = true
mermaid = true
outdate_alert = false
outdate_alert_days = 120
display_tags = true
truncate_summary = false
+++

# Intro & Determination
Hello!
In this short blog post, I will show you how I implemented a custom GPU based linear allocator for one of my unity projects involving custom terrain generation and GPU instancing. One of the main reasons one would want such a GPU allocator is so that they can allocate temporary memory on the GPU without having to readback to the CPU to allocate buffers there. This is because reading back to the CPU to create buffers not only adds a lot of latency, but is also very slow, so when experimenting I decided to leave everything up to the GPU and keep as little as possible on the CPU.

I have implemented this sorts of GPU allocator twice already, once for my custom game engine, and once for the unity project I work on. Both of these projects needed some sort of GPU allocation algorithm to allocate GPU memory with minimal read back to the CPU. 

# The Naive Method
The first method I thought of to implement such a system would be to simply read back the temporary memory data to the CPU, and manually copy it to a free spot within the big buffer. However, even before implementing such a thing, I knew that it would be *extremely* bad for performance due the latency and limited transfer bus for GPU -> CPU readback. Not only that, but it would seem wasteful to generate a lot of data extremely fast on the GPU to then read it back on the slow CPU to be able to upload it once again to the GPU. It just didn't seem like a very fast method to achieve what I wanted at respectable framerates, which is why this pushed me to implement my own custom linear GPU allocator

# Part 1: The Main Idea
The big idea of such an allocator would be that it would be able to copy my temporary memory (from my temp buffer) to a permanent memory **without** overlapping or overwriting any data already present in the permanent buffer. It should also be possible to *know* where it copied previous blocks so that we are able to remove them afterwards (analogous to deallocating memory). 

So these are the following restrictions for such an allocator:
* **Most of the work on the GPU with minimal data transfer to CPU**
* **Relatively fast to allocate new memory chunks**
* **Both GPU and CPU should know what memory chunks are currently in use**
* **No overlapping / overwriting of old chunks**

So with these restrictions I've set out to implement a simple linear allocator with a CPU/GPU bitset to allow us to know what chunks are currently in use (where each "chunk" is just a region of 4kb memory, so that we don't need to allocate each byte individually, which would be horrendously slow).

In my most recent implementation of this algorithm, I have 4 distinct compute shaders that run sequentially to ensure proper memory allocation:

1. Free memory chunk finder (find a free memory chunk linearly)
2. Memory chunk copy (copy temp memory to perm memory block)
3. Memory chunk removal (deallocating)

# Pre-requesites for dynamic allocation
Before we even tackle dynamic allocation, we must understand that this method of allocating memory is limited by the amount of VRAM that our GPU has. In my case, I have a GTX 1050 mobile with 3GB VRAM, and the maximum buffer size I can possibly allocate in Vulkan is 2GB, however after a lot of testing I've figured out that it is best to allocate 4 or less buffers of 512 megabytes. In my Unity implementation for example, I just have one big buffer that contains a few megabytes of data since prop generation doesn't take that much memory.

# Part 2: GPU Parallel min search
The first step in allocating memory on the GPU is to find a free block of memory. In my implementations I did this both on the CPU and on the GPU so that I can keep a copy of my indices on the CPU since I will need to be able to remove (aka deallocate) chunks of memory later on.

To store what chunks are currently allocated and in use I simply make use of dynamically allocated bitset where each bit repsents a "sub-allocation". (*in my unity terrain generator each bit simply represents a prop since I haven't implement sub-allocations yet*). This allows me to 

# Part 3: Temp memory -> Perm memory copy


![Allocator Debug View in cFlake Engine](/terrain_allocations_debug.png)
