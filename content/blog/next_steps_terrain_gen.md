+++
title = "Unity Voxel Terrain Generator: Next Steps"
date = 2024-09-24
draft = true

[taxonomies]
categories = ["Unity"]
tags = ["unity", "graphics", "procedural"]

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

In this short blog post I'll talk about the current problems / missing features with my ``Unity Voxel Terrain Generator`` package and what are the next steps I'll be taking to improve on it

# Summary Of Problems
* Octree pain in the butt but also very nicely
* Cant used data used for children to calculate parents' data (just doesn't work, I tried it. We must recompute it)
* World edits are handled using a "delta" system meaning that we do not know the underlying voxel data
* Editing cannot occur during generation (reason: me being lazy)
* Meshes require n+3 voxels to generate a mesh of size n. Causes/forces us to:
    * Weird scaling issue with problems occuring at boundaries
    * Textures to be non-power of two
* Chunks are independent making SN kinda stoopid
* Async readbacks for all chunks (even though we only need it for those that will be modified)
* Impostors have incorrect world normals. Can be fixed by capturing at different camera angles and blending
* Floating impostors due to low-resolution segments voxel grid

Definitions:
**SN**: Surface Nets
**MC**: Marching Cubes
**DC**: Dual Contouring 
**Octree**: 3D tree structure where each node can have up to 8 children
**LOD**: Level of Detail. Used in video games to lower the 



# Octree/chunk stuff

## "Lazy" octree
During the development of this package, I've hit numerous roadblocks simply because I was using something that could be called a "lazy" octree. Either it be editing or prop generation, so it's been pretty annoying so far.

Currently, whenever I generate the terrain, I first generate an octree that incrementally increases the resolution of chunks that are nearby the player camera (LOD). The problem is, the terrain generator only generate the voxel data for chunks *within* the octree at a specific "snapshot" in time.
So for example, any chunks that that are currently present in the terrain contain their unique voxel data that *they* generated. If I were to unload those chunks by getting further away from then and load them back it by getting closer, the voxel generate would need to recompute that voxel data twice.
The octree is "lazy" in the sense that it only computes what is necessary for the current chunks at the current instant, it doesn't cache or store anything in lower LOD chunks that could aid generation later on.

### Recomputation

### Modifying Lower Lod Chunks
This is a problem because if we want to, for example, modify a chunk that isn't at level 0, we'd need to keep some sort of "incremental" data that we can apply to both the chunk at level 0 and higher LOD-leveled chunks (which is what we are doing at the moment actually).

### Time Independent World Scale Edits
Another con that comes with an octree system (or at least, not having all the terrain voxel data loaded in) is that we can't have any "world scale" edits occurring:
Like for example, let's say you're trying to make some sort of wind edit that modifies the underlying terrain by chipping away at the least dense points based on some wind direction. With our current system you can try getting around some of the limitations by implementing a "time-since-last-updated" value and implementing the edit as a measure of time instead, but it would still be pretty hacky. We don't have all the chunks at the highest resolution loaded in to be able to modify their voxel values freely like that, and even we were to force load them for such an edit, that completely defeats the benefits of an octree during edits (albeit that isn't as much of as I make it sound to be) 

## Mutually Exclusive Editing & Generation
This was mostly cause I was fucking stupid and told the editing script to always halt until all chunks are generate and vice versa. This caused a bunch of stalls in the pipeline that weren't actually needed. The main reason I even implemented this was because if the user was editing a chunk and the terrain generator had to unload it / modify its voxel data, the job system would freak out because the editing system and terrain generation system don't really depend on each other and always assume that NO other operations are currently executing, just to make code logic a bit easier to handle.

A smarter way to tackle this would just to keep track of the chunks that are actively being modified and wether or not we can safely modify / unload them. This would allow you to modify chunks that you *are* allowed to modify (the ones closest to the player anyways) and avoid having to stall ALL chunk operations.

Basically it was just too conservative, which isn't necessary in some cases where the sub-managers don't have anything that depend on each other.

## Independent Chunk Generation

## Scaling "n+3" Issue

# Meshing Stuff


## Surface Nets Chunk Gaps
The current meshing algorithm (aka the algorithm that I use to convert from 3D density texture --> mesh with vertices and triangles) is SN, which is like a cousin to DC
In of itself, this isn't a problem, and it's actually a good thing because SN is usually faster than something like DC or MC, but it has a pretty major flaw in my current system:

it causes major gaps in the terrain when used with an octree or dynamic resolution system like mine (MC also create gaps but they're easily "fillable" and way less noticable that SN's gaps)


Now you might say "ohhh well marching cube also produces these gaps jed you're fucking stupid". Well, yes you'd be right in that sense but at least with the marching cubes algorithm you can use something like skirts or transvoxels.

Just look at the horrible skirts example below:
![](/bad_skirts.png)

As you can see, it looks like the terrain has a big flat area in that region, even though the neighboring terrain is mostly smooth! How can this be?? This is because my implementation of skirt just *fucking* **sucks!!**

The current implementation is really hacky, and it gives really really bad worst-case results, but in the average case it's *meh*. The underlying code just clamp/stretches the vertices to the edges, no matter if they are extra verts needed for skirts or not. This really isn't the way of doing things, especially with surface nets (since DC in of itself is mostly used with multi-resolution meshes and with octrees. I've just been off-putting actually implementing that)

Two solutions to this problem:
* Actually implementing proper stitching support like in multi-resolution DC
    * Want to avoid this at all costs because having to implement inter-chunk DC would require inter chunk dependencies which would be a pain to keep track of
* Find an algorithm that has edge vertices like marching cubes so that skirts work nicely (more inclined into doing this)
    * I recently found a paper online that implements the second method, I sure hope it's *not* using chunk inter-dependency otherwise I will be very sad :(, but yea I will try implementing that and finally get rid of this pesky issue once and forall.


## Slow/Stuttery/Limited Async Readbacks
As the header states, we don't even need to do async readback to generate the mesh on the CPU! We can duplicate the meshing algorithm on the GPU and execute it directly after the voxelization step. This is what I did for the terrain generator in cFlake engine and it was really REALLY fast! (like, a few milliseconds just to mesh a 64x64x64 volume. Surface Nets can really take advantage of the parallel architecture of GPU compute shaders).

Actually, I could've easily written the mesh portion of the terrain generator on the GPU, but I decided not to. My reasoning was, that since we'd have to do a bunch of terrain edits (something that is inherently executed on the CPU using the Job System instead of compute shaders), it would be easier to just have the meshing algorithm on the CPU ready to re-mesh chunks whenever their voxel data is modifed. 

This works fine for the most part, but it could be optimized even more for chunks that we *know* will never get modified, like those low resolution chunks far away from the player. For those, we can avoid reading back voxel data to the CPU and just mesh the chunk on the gpu instead. This avoid an unecessary async readback, which avoids stutters

In of themselves async readbacks don't cause stutters / stalls (since well, they're asynchronous), but when you make more than 1 async callback per frame it starts to "feel" heavy as in you can start feeling that the FPS drops (which happens anyways by a slight amount if the CPU is always waiting for the GPU). Anyways, I feel like we could save a bit of the CPU <-> bandwidth by avoiding that async readback.

Basically, this was how the pipeline was working:
{% mermaid() %}
%%{ init: { 'flowchart': { 'curve': 'linear' } } }%%
flowchart LR
subgraph A[GPU]
S[GPU Voxel Data]
end
subgraph B[GPU <==> CPU]
M[Async CPU Readback]
end
subgraph C[CPU]
G[Mesh Generation]
R[Dispatch Rendering Jobs]
end
subgraph A[GPU]
RD[Actually rendering chunks]
end
S --> M
M --> G
G --> R
R --> RD
{% end %}

We're just causing a slow-down in the pieline by having to readback the data to the CPU. It'd be much nicer if we can just avoid doing that (which is just what indirect rendering and indirect compute commands are for, which I've used before in a terrain generator for my custom engine).  

# Prop stuff
## Incorrect Impostor World Normals
{{ figure(src="/awful_impostors.png", caption="You can kinda notice the difference in tree props that are real 3D objects and which ones are 2D impostors") }}

<div class="image-row">
{{ figure(src="/zoomed_impostor.png", caption="Impostors like this one kinda feels flat") }}
{{ figure(src="/zoomed_real.png", caption="This one pops out more, simply because it's actually 3D") }}
</div>

As you can see, from a close distance, you can clearly tell what props are the impostors and which ones are the actual 3D gameobjects.
This is because the normal information of the props does't really match up at *all* angles and perspectives. When we take a snapshot of the impostor diffuse/normal/mask textures, it's only at one camera angle, and as such, these values don't really match up when you rotate the impostor around

A smarter way to tackle the problem would be to take multiple pictures at different camera angles around the source prop. This would match up more closely to how the props are viewed in the game directly. There is actually a [blog post](https://shaderbits.com/blog/octahedral-impostors) about about an implementation of such impostors into UE4 and the Fortnite map, really cool stuff (that's where I got my idea from anyways).

Updating the current impostors to use this new technique shouldn't be too too hard, I'll just need a way to pack such information inside my already array based system (due to having support for multiple variants for each prop). Maybe I could make use of image stitching or something similar.
Also want to read up on other ways to handle impostor rendering, maybe through other means like raymarching or something like that. Could be interesting as well.

## Floating Props
This issue occurs because our props and their respective impostors are generated using a completely separate voxel grid. The reason I did that was because if I were to use the chunks' voxel grid, the spawn resolution of the props would decrease as the camera got further away from the chunks (due to the LOD system). However, you can't just compute ALL the voxels as that's a complete waste of resources, it's as if you never had an octree system to begin with!

I need to find a way to "attach" surface props (props that are spawned at the surface of something, like trees or rocks) to the closest surface whenever new chunks are generated. And I've got to remember that I have to update the GPU buffer as well since there are some props that aren't spawned in as prefabs (like instanced props).

I guess we can just make some sort of compute shader that does ray-tracing directly on the GPU based on a chunk's voxel data / triangles? And then we could snap the props to those positions using a secondary *per-segment* buffer that is automatically modified whenever its underlying chunks are modified (or when the resolution changes or something like that).

## Inefficient Impostor Culling & Limited Prop Count
Currently, prop culling is done using a really *really* stupid heuristic. It's using a simple dot product between the camera forward vector and the vector pointing from the player to from the impostor 

# Other

## Missing multiplayer support
Technically this should be really simple. As long as we can prove that the compute shaders that we used for voxel/prop generations give the same result on different machines on the same platform we should be able to implement multiplayer support pretty easily 
As long as compute shaders are stable enough (aka same seed -> same result, roughly) across machines then we can do the following to implement multiplayer support:

### Voxel/Prop Generation
* Server/client duplicated world generation (every client runs the terrain generator locally)
* Shared world seed with every client that joins a server

### Terrain Edits
* Client sends edit itself to server
* Server broadcasts edit to every client
* Periodically send whole world delta data?
* Possibly do delta compression for specific edit types 

### Prop Data
* Serialize prop data, send to server to broadcast to every client
*  When a client receives from server, update their prop data accordingly:
    * If the prop wasn't loaded on the client, this just stores that data in the local save file instead (since that's where prop data for unloaded props is stored). Next time the user loads that prop, the new data will be fetched from their local file directly

## Saving system bad compression ratio
TODO