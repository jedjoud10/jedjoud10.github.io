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
10) Doesn't support ediiting whilst generating new chunks (implemented as a way to avoid job dependency errors, needs to be smarter)

Most of these issues could be linked to these two main reasons: The fact that I use a "lazy" octree, and the fact that I use surface nets as my meshing algorithm

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