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

In this short blog post I'll talk about the current problems with my ``Unity Voxel Terrain Generator`` package and what are the next steps I'll be taking to improve on it
Here are the main issues at the moment (directly copied from the github repo)

* Still riddled with bugs
    1) Editing terrain sometimes leaves gaps
    2) Prop generation sometimes breaks out of nowhere
3) Terrain chunk scheduling is non-conservative. Always over-estimates the amount of chunks actually containing terrain
4) Bad performance when editing large voxel/dynamic edits (due to the dupe-octree nature of voxel edits)
5) Bad memory consumption / saved world size due to dupe-octree
6) Slow async GPU readback which causes frame time spikes when there is more than 1 request per frame
7) Billboarded prop normals don't seem to match up with their gameobject counterpart (seem fine at a distance, only noticeable at some lighting conditions)
8) Floating terrain segments (could fix by running a flood fill and seeing the parts that aren't "connected")
9) Floating props (due to low-resolution segment voxel grid)
10) Doesn't support editing whilst generating new chunks

Most of these issues could be linked to these two main reasons: The fact that I use a "lazy" octree (and just generally using an octree), and the fact that I use surface nets as my meshing algorithm. I'll also explain the separate issues in more detail afterwards.

## "Lazy?" octree
Let me first explain what I mean by "lazy" octree. Currently, whenever I generate the terrain, I first generate an octree that gets higher and higher quality around the player camera (which gives us the LOD effect). This, in of itself, is a completely fine way of handling LODs. However my terrain generator only generate the voxel data for chunks *within* the octree. So for example, any chunks that that are currently present in the terrain contain data that *they* generated. For example, if I were to get further away from said chunk and then get close again, that voxel data would need to be recomputed!

What I'm trying to say is, each octree chunk only contains the voxel data to generate its mesh and **only that**. This is a problem because if we want to, for example, modify a chunk that isn't at level 0, we'd need to keep some sort of "incremental" data that we can apply to both the chunk at level 0 and higher LOD-leveled chunks. In any case, the LOD system that I'm using at the moment is a giant pain in the butt and it's forced me to do "hacky" things to make them work with what we needed for *Hypothermia*. 

Another con that comes with an octree system (or at least, not having all the terrain voxel data loaded in) is that we can't have any "world scale" edits occuring. Like for example, let's say you're trying to make some sort of snow edit that accumulates snow over time in one specific region. With our current octree... you simply can't do that! We don't have all the chunks at the highest resolution loaded in to be able to modify their voxel values like that! And even i we force them to load for such an edit, that completely defeats the purpose of having the octree in the first place! 

If I, for example, were **not** to use an octree system and rather a fixed grid, I would be able to fix at least problems 1, 4, 3, and 9. Heck, if I weren't using an octree I wouldn't even need to complain about the Surface Nets Algorithm in the next part but whatever.

It's really really powerful because it can handle gigantic terrain at acceptable framerates, but it's also a pain to deal with, especially when it comes to terrain edits that don't affect the highest level chunks, forcing us to hack a way to modify the chunks up the chain as well.

After checking out the pictures from [this beautiful post](https://ngildea.blogspot.com/2014/09/dual-contouring-chunked-terrain.html) again, I realized that there *is* a way to handle LOD without using an octree... just have fixed size chunks reduce in size the further they get! Cool idea that I completely overlooked...

## Surface Nets Shenanigans
Second issue is that we're using the surface nets algorithm for generating our meshes. This, in of itself is actually a bonus, since the surface nets algorithm is usually faster than something like dual contouring or marching cubes, but it has a pretty major flaw when used with an octree or dynamic resolution system like this: it causes major gaps in the terrain. Now you might say "ohhh well marching cube also produces these gaps jed you're fucking stupid". Well, yes you'd be right in that sense but at least with the marching cubes algorithm you can use "proper" skirts. Just look at the horrible skirts example below:
![](/bad_skirts.png)

As you can see, it looks like the terrain has a big flat area in that region, even though the neighbouring terrain is mostly smooth! How can this be?? This is because my implementation of skirt just fucking sucks!! The current implementation of skirts that I use with my surface nets algorithm at the moment is really hacky, and it gives really really bad results. I basically just "clamp" the vertices to the edges, no matter if they are extra verts needed for skirts or not. This really isn't the way of doing things, especially with surface nets (since its cousin, Dual Contouring, is mostly used with multi-resolution meshes and with octrees. I've just been off-putting actually implementing the octree part of it)

There are two solutions to this problem:
* Actually implementing proper stitching support like in multi-resolution Dual Contouring
* Find an algorithm that has edge vertices like marching cubes so that skirts work nicely (more inclined into doing this)

The reason I'm more inclined into using skirts is that they're chunk independent, so I can still keep the "parallelism" (like I had any, lol) of generating multiple chunks without one having to depend on another to complete generation.

## Impostor normals don't match up with their game object counterpart
![](/awful_impostors.png)

As you can see, from a close distance, you can clearly tell what props are the impostors and which ones are the actual 3D gameobjects. This is because the normal information of the props does't really match up. I think I have an idea to fix this, but it would require me to re-write how impostors are drawn. Basically, how I handle it at the moment is I just take a *single* picture of each prop variant and use their world normals as is. A smarter way to tackle the problem would be to take multiple pictures, at different camera angles around the source prop. This would match up more closely to how the props are viewed in the game directly. There is actually a [blog post](https://shaderbits.com/blog/octahedral-impostors) about about an implementation of such impostors into UE4 and the Fortnite map, really cool stuff (that's where I got my idea from anyways).

Updating the current impostors to use this new technique shouldn't be too too hard, but I'

## Floating props (due to low-resolution segment voxel grid)
This issue occurs because our props and their respective impostors are generated using a completely separate voxel grid. The reason I did that was because if I were to use the chunks' voxel grid, the spawn resolution of the props would decrease as the camera got further away from the chunks (due to the LOD system). However, you can't just compute ALL the voxels as that's a complete waste of resources, it's as if you never had an octree system to begin with!

I need to find a way to "attach" surface props (props that are spawned at the surface of something, like trees or rocks) to the closest surface whenever new chunks are generated. And I've got to remember that I have to update the GPU buffer as well since there are some props that aren't spawned in as prefabs (like un-interactables).

I guess we can just make some sort of compute shader that does ray-tracing directly on the GPU based on a chunk's voxel data / triangles? And then

## Slow async readbacks, or well, the fact that we do async readbacks to begin with
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

We're just causing a slow-down in the pieline by having to readback the data to the CPU. It'd be much nicer if we can just, avoid doing that (which is just what indirect rendering and indirect compute commands are for, which I've used before in my terrain generator).

## Doesn't support editing whilst generating new chunks
This was mostly cause I was fucking stupid and told the editing script to always halt until all chunks are generate and vice versa. This caused a bunch of stalls in the pipeline that weren't actually needed. The main reason I even implemented this was because if the user was editing a chunk and the terrain generator had to unload it / modify its voxel data, the job system would freak out because the editing system and terrain generation system don't really depend on each other and always assume that NO other operations are currently executing, just to make code logic a bit easier to handle.

A smarter way to tackle this would just to keep track of the chunks that are actively being modified and wether or not we can safely modify / unload them. This would allow you to modify chunks that you *are* allowed to modify (the ones closest to the player anyways) and avoid having to stall ALL chunk operations just in case.

Basically it was just too conservative, which isn't necessary in our case.


