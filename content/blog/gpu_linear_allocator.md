+++
title = "GPU Compute-based Linear Memory Allocator"
date = 2024-03-01
draft = false

[taxonomies]
categories = ["Compute", ]
tags = ["game-dev", "ecs", "custom-engine"]
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

To store what chunks are currently allocated and in use I simply make use of dynamically allocated bitset where each bit repsents a "sub-allocation". (*in my unity terrain generator each bit simply represents a prop since I haven't implement sub-allocations yet*). This allows me to run a fast algorithm to detect what regions in memory are not being in use and ones that we could allocate. In both my implementations, I created a compute shader that handles this task, and this is its source code and how it works:

```glsl
#version 460 core
layout (local_size_x = 128, local_size_y = 1, local_size_z = 1) in;

// Spec constants for sizes
layout (constant_id = 0) const uint sub_allocations = 1;
layout (constant_id = 1) const uint vertices_per_sub_allocation = 1;
layout (constant_id = 2) const uint triangles_per_sub_allocation = 1;

// Each group consists of 128 sub allocations
const uint sub_allocation_groups = sub_allocations / 128;
const uint sub_allocations_per_sub_allocation_group = 128;

// Sub allocation chunk indices
layout(std430, set = 0, binding = 0) buffer SubAllocationChunkIndices {
    uint[sub_allocations] data;
} indices;

// Allocation offsets
layout(std430, set = 0, binding = 1) buffer FoundOffsets {
    uint vertices;
    uint triangles;
} offsets;

// Atomic counters
layout(std430, set = 0, binding = 2) readonly buffer Counters {
    uint vertices;
    uint triangles;
} counters;

// Must be shared for atomic ops between groups
shared uint chosen_sub_allocation_index;

// Sub-allocations:         [-1] [-1] [3] [3] [3] [-1] [2] [2]
// Sub-allocation groups:   [                                ]
// Dispatch invocations are sub allocation groups
void main() {
    if (gl_GlobalInvocationID.x > sub_allocation_groups) {
        return;
    }

    // Checks if we are within a free region or not
    bool within_free = false;

    // Keeps count of the number of empty sub allocations that we passed through 
    uint free_sub_allocations = 0;

    // Length of what we need to find sub allocs for
    uint vertices = counters.vertices;
    uint triangles = counters.triangles;

    // Doesn't really matter since we can calculate it anyways 
    uint vertex_sub_allocation_count = uint(ceil(float(vertices) / float(vertices_per_sub_allocation)));
    uint triangle_sub_allocation_count = uint(ceil(float(triangles) / float(triangles_per_sub_allocation)));
    uint chosen_sub_allocation_count = uint(max(vertex_sub_allocation_count, triangle_sub_allocation_count));
    uint reason = vertex_sub_allocation_count > triangle_sub_allocation_count ? 2 : 1; 

    // Temp values for now 
    uint temp_sub_allocation_index = 0;
    uint temp_sub_allocation_count = 1;

    uint invocation_local_chosen_sub_alloction_index = 0;

    memoryBarrier();
    barrier();

    // If we are the first group, update temporarily
    if (gl_GlobalInvocationID.x == 0) {
        atomicExchange(chosen_sub_allocation_index, uint(-1));
    }

    memoryBarrier();
    barrier();

    // Find a free memory range for this specific sub-allocation group
    uint base = gl_GlobalInvocationID.x * sub_allocations_per_sub_allocation_group;
    for (uint i = base; i < (base + sub_allocations_per_sub_allocation_group); i++) {
        bool free = indices.data[i] == uint(-1);
        
        // We just moved into a free allocation
        if (!within_free && free) {
            temp_sub_allocation_index = i;
            temp_sub_allocation_count = 0;
            within_free = true;
        }

        // We stayed within a free allocation
        if (within_free && free) {
            temp_sub_allocation_count += 1;
        }
        
        // If this is a possible candidate for a memory alloc offset, then use it
        if (within_free && temp_sub_allocation_count >= chosen_sub_allocation_count) {
            atomicMin(chosen_sub_allocation_index, temp_sub_allocation_index);
            invocation_local_chosen_sub_alloction_index = temp_sub_allocation_index;
            break;
        }

        // Update to take delta
        within_free = free;
    }

    memoryBarrier();
    barrier();

    // Only let one invocation do this shit
    if (gl_GlobalInvocationID.x != 0) {
        return;
    } 

    memoryBarrier();
    barrier();

    // After finding the right block, we can write to it
    for (uint i = chosen_sub_allocation_index; i < (chosen_sub_allocation_index + chosen_sub_allocation_count); i++) {
        indices.data[i] = reason;
    }

    // Offsets that we will write eventually
    offsets.vertices = chosen_sub_allocation_index * vertices_per_sub_allocation;
    offsets.triangles = chosen_sub_allocation_index * triangles_per_sub_allocation;
}
```

That's quite a lot of code so lemme break it down for you. Basically, there are 3 steps that occur within this ``find.comp`` compute shader
1. Get required memory block size to allocate
2. Initialize min parallel search
3. Find lowest chunk index to reduce fragmentation (*fun part*)

## Get required memory block size
What this basically means is that we just need to know how *much* memory we should allocate. This should be a trivial matter since we are using this dynamic allocate for a reason; allocating dynamic memory!
So this step, in most if not all cases, is already done. In my code it's a bit more complicated since I actually allocate two buffers at once (*for my vertices and triangles*) but that's basically what this code does.

```glsl
// Length of what we need to find sub allocs for
uint vertices = counters.vertices;
uint triangles = counters.triangles;

// Doesn't really matter since we can calculate it anyways 
uint vertex_sub_allocation_count = uint(ceil(float(vertices) / float(vertices_per_sub_allocation)));
uint triangle_sub_allocation_count = uint(ceil(float(triangles) / float(triangles_per_sub_allocation)));
uint chosen_sub_allocation_count = uint(max(vertex_sub_allocation_count, triangle_sub_allocation_count));
uint reason = vertex_sub_allocation_count > triangle_sub_allocation_count ? 2 : 1; 
```

## Initialize min parallel search
Just initialize some global and thread local variables to make use of GPU parallelisation. Basically the following lines

```glsl
// Must be shared for atomic ops between groups
shared uint chosen_sub_allocation_index;

// Temp values for now 
uint temp_sub_allocation_index = 0;
uint temp_sub_allocation_count = 1;

uint invocation_local_chosen_sub_alloction_index = 0;

// Checks if we are within a free region or not
bool within_free = false;

// Keeps count of the number of empty sub allocations that we passed through 
uint free_sub_allocations = 0;

memoryBarrier();
barrier();

// If we are the first group, update temporarily
if (gl_GlobalInvocationID.x == 0) {
    atomicExchange(chosen_sub_allocation_index, uint(-1));
}

memoryBarrier();
barrier();
```

## Find lowest chunk index (actual search algo)
This is the actual search algorithm. It is very stupid and naive, but it seems to work at reasonable performance in my engine. What I do is simply loop over all the chunks in multiple GPU invocations as to look for an empty chunk of memory in parallel.

```glsl
// Find a free memory range for this specific sub-allocation group
uint base = gl_GlobalInvocationID.x * sub_allocations_per_sub_allocation_group;
for (uint i = base; i < (base + sub_allocations_per_sub_allocation_group); i++) {
    bool free = indices.data[i] == uint(-1);
    
    // We just moved into a free allocation
    if (!within_free && free) {
        temp_sub_allocation_index = i;
        temp_sub_allocation_count = 0;
        within_free = true;
    }

    // We stayed within a free allocation
    if (within_free && free) {
        temp_sub_allocation_count += 1;
    }
    
    // If this is a possible candidate for a memory alloc offset, then use it
    if (within_free && temp_sub_allocation_count >= chosen_sub_allocation_count) {
        atomicMin(chosen_sub_allocation_index, temp_sub_allocation_index);
        invocation_local_chosen_sub_alloction_index = temp_sub_allocation_index;
        break;
    }

    // Update to take delta
    within_free = free;
}
```

So within the many parallel GPU invocations, I do a sequential linear search to find a region of blocks of ``n`` size so that we can use it for our permanent memory allocation

```glsl
if (within_free && temp_sub_allocation_count >= chosen_sub_allocation_count) {
    atomicMin(chosen_sub_allocation_index, temp_sub_allocation_index);
    invocation_local_chosen_sub_alloction_index = temp_sub_allocation_index;
    break;
}
```

these lines are really important, as they avoid one big problem of such a system; ``fragmentation``
Since we look for empty chunks in parallel, there are many cases in which we *find* many empty chunks candidates, but they're all scattered around the memory heap. So I implement this ``atomicMin`` to find the *lowest* index and the region that is closest to the start of the GPU buffer. This reduces fragmentation which allows us to allocate memory more efficiently. 

Here's a picture of the debug view of the allocate in cFlake engine. As you can see, most white chunks are at the start of the buffer, and there aren't any at the end of the buffer, which is what we want.
![Allocator Debug View in cFlake Engine](/terrain_allocations_debug.png)

# Part 3: Temp memory -> Perm memory copy
{% note(header="Edit") %}
Ok I just checked the Vulkan extension lookup (March 24th 2024) for my GPU and I found that there is a ``VK_NV_copy_memory_indirect`` extension that does the stuff that I will explained below, without actually having to implement it in a custom compute shader. You could probably skip the following text and use the extension instead but at the time of writing this for my engine I didn't know this extension was available (and even if it was, I wouldn't be able to use it since I used wgpu, unless I do some sort of Vulkan/wgpu interop) 
{% end %}

This is actually the simplest part of the allocator, and its role is to simply copy the temporary memory we wish to allocate into the chunks we just allocated. Simply like ``memcpy``-ing into a ``malloc``. In of itself, this task is extremely simple; just run a compute shader with a specific size that will copy the memory from one region to another region; the tricky part is that you must decide on the size of the dispatch call for such a compute shader. In all my previous implementations, I went with the naive but easy way of just calling a few dispatches with massive loops inside the compute shader itself, however one could make this a lot more optimized by using indirect dispatch instead but I haven't found any bottlenecks yet so I'm not going to bother.

# Conclusion & Results
So, as far as results go, this whole allocator works pretty darn nicely. I haven't profiled it but I can assume it's relatively fast. The combination of using bitwise ops and having a "minimum" allocation size (governed by the sub-allocation size) keeps things relatively simple too. 

As I said before, I've used this in both my custom rust game engine [cFlake](https://github.com/jedjoud10/cflake-engine) for terrain mesh generation and in my [Voxel Unity Package](https://github.com/jedjoud10/VoxelTerrainGenerator) to handle prop generation, and in both cases it worked pretty flawlessly.

The only problems I encountered with this if I recall where sometimes data would overwrite / overlap itself when there is no more space to write to (this mostly happened in my unity prop generator). This happens because I have no fallback that gives out an error in that implementation, but I did in my custom engine I think.