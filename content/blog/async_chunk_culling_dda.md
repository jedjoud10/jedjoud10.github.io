+++
title = "Async Entity Culling using Software DDA"
date = 2025-08-25
draft = true

[taxonomies]
categories = ["Random"]
tags = ["c#", "voxels", "optimization"]

[extra]
lang = "en"
toc = true
comment = false
copy = true
math = false
mermaid = false
outdate_alert = false
outdate_alert_days = 120
display_tags = true
truncate_summary = false
+++

# Abstract
Usually, when you try to render lots of things on screen, you notice that a lot of the entitires in the world aren't actually visible to the camera that's rendering them. Either because the entity is outside the view frustum of the camera (ex: behind it), or because it is being occluded by another entity in front of it.

We usually want to avoid rendering (or **culling**) these entities to lighten the load on the GPU, as it would have rendered something that would not be visible to the user at the end. This is where culling comes in. There are two types of culling; frustum and occlusion.
- **Frustum** culls entities that are outside the bounds of the camera frustum.
    - *Implementation*: This is easy to do with 6 simple plane distance checks generated from the camera frustum to see if the entity is fully outside the frustum. This can be done on the CPU before sending the data to the GPU, or even indirectly on the GPU.
- **Occlusion** culls entities that are hidden away completely by other entities
    - *Implementations*: 
        - *Naive AABB overlap check*: Causes lots of false-positives, as the AABB is a **bigger** bound than the underlying mesh it is representing. you need a *minimum* AABB bound, but that would not help much either as most meshes are concave, not convex
        - *My Implementation*: Voxel raytracing. 

In both cases, you want to make sure that your culler is non-conservative to avoid false positives (which would cause visual artefacts). It would be better to not-cull something that is visible than to cull something that is invisible.

I'll be showcasing my current implementation of occlusion culling that I implemented in **Unity ECS** to cull occluded entities and chunks.  

# Implementation
Since the entirety of the terrain generator is built using voxels as the root, then doing occlusion culling *using* the generated voxels will come in handy, as we don't have spend time voxelizing our meshes over and over again.

