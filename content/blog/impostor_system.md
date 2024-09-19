+++
title = "Automaticaly captured multi-draw instanced impostors in Unity"
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
In this short blog post I'll demonstrate how I was able to draw multiple thousand impostors and their variants without having to manually generate their textures! My system uses an "auto-capture" method that automatically find out all combinations of variants and camera rotations / positions to be able to generate these values at the required angles. 

Here's a list of vocab stuff that I used:
* Prop: General terrain prop since my system is based on terrain generaiton. This is just a game object that will then be converted to an impostor
* Impostor: Fake prop that is rendered only using a billboard and a few textures that are automatically captured
* Diffuse Texture: Contains only the color data of something to be rendered
* World Normal Texture: Contains the **world** normal data of something to be rendered. Different than normal maps since those store their data in tangent space.

## Big stuffs
This system can be split up to 3 **main** parts:
1. Capturing required texture data during initialization for the required mesh variants
2. Creating buffers that contain the transform and variant types for each impostor type. 
3. Rendering the impostors using a custom shader and using Unity graphics commands directly.

{% note(header="Note") %}
In my terrain package, creating the compute buffers for impostor data and filling them up is done automatically. 
However, I will explain what steps you should take in order to implement this for any type of system
{% end %}

# Part 1: Capturing required data
This is the most interesting and most important step of the system, as this is where we will decide the main factors that affect how our impostors will look like:
* How the required render textures are captured
* What parameters we should use to capture them (camera rotation, distance, scale)

For my current implementetion, I've settled for only using albedo and normal map data from my real meshes (however you could make this work with an AO map really easily down the line). Since I am using the Unity game engine, I have written all the required parameters and required meshes down into multiple scriptable objects that I feed to my capture script at the staart of the game.
Here's a screenshot of what impostor data you could give to it:

![](/prop_data.png)

So the main things to keep in mind here are the following data entries:
* Variants: A list of multiple prefabs and billboard capture settings (camera transform and distance)
* Prop spawn behavior: How props should handle rendering
* Billboard capture settings:
    * Texture width & height: **EXTREMELY** important! This depicts the size of the textures that would be used for billboard rendering
    * Filter mode and mipmap mode: Self explanatory. Sometimes I found mipmaps to be too blurry so I added this setting to just disable them completely
* GPU Instance Rendering: Very useful as well since this allows us to enable or disable shadows for the impostors (using a little shadertoy hack I found out that I will explain later). Also allows us to lock the rotation in the Y direction. Very useful for trees as you wouldn't want them to really *look* at the camera, as pop-in would become really visible when you look at it from above / below.

I capture the diffuse / world normal data using a custom shader that I apply to the spawned prop mesh