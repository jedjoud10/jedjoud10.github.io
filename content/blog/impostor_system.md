+++
title = "Unity Voxel Terrain Generator: Imposters Among Us"
date = 2024-09-15
draft = true

[taxonomies]
categories = ["Unity"]
tags = ["unity", "graphics", "shader-graph"]

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

# Intro

In this short blog post I'll demonstrate how I was able to draw multiple thousand trees using ``billboarded impostors``, a technique that uses ``billboards`` that face the camera, alongside some shader tricks to make them look good.

My system uses an **"auto-capture"** system that automatically captures all the necessary textures *in the editor* which means that I don't need to manually create these textures myself. Also avoids capturing the prop textures at runtime which helps with initialization times, since it is a fairly expensive process. 

Here's some definitions first
* Prop: General terrain "object". Just a game object that is spawned during chunk generation that will then be converted to an impostor at further distances
* Impostor: Basically, billboard with pre-generated textures at each viewing angle. This gives the billboard some "depth"
* Diffuse Texture: Contains only the ``color data`` as RGBA (alpha used as mask)
* World Normal Texture: Contains the ``world normals`` (different than normal maps since those store their data in ``tangent space``) 
* Mask Texture: ``ARM`` based mask masp (``Ambient Occlusion``, ``Roughness``, ``Metallic``) stored in the R,G,B channels of the texture in that order.

## Big stuffs
This system can be split up to 3 **main** parts:
1. Capturing required texture data in the editor
2. Creating GPU buffers and texture arrays. 
3. Rendering the impostors.

# Part 1: Capturing required data
This is the most important step of the system, as this is where we will decide how the impostor textures are captured.
My system uses an orbiting camera that takes ``32`` snapshots of the prop at different angles. This allows me to sample these textures at runtime

## C# Prop Code
## C# Capture Code
## Shader
# Part 2: Creating prop graphics buffers

# Part 2: Rendering the impostors
## Compute Based Dot Product (stupid) Culling
## Shader graph hack for instanced indirect