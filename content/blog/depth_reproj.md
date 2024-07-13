+++
title = "Reprojecting depth values as an acceleration structure for ray-marching"
date = 2024-05-21
draft = true

[taxonomies]
categories = ["Compute Shaders"]
tags = ["c#", "openTK", "openGL", "glsl", "graphics-dev"]

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

# Yapping
In this short blog post, I will show you how I managed to use reprojected last frame depth information as an optimization for my C#/OpenGL voxel ray-marcher. So, for the past few months, I've been working on this super tiny voxel raymarcher written in OpenGL and OpenTK. I finally decided to give in into the hype train after being convinced after watching custom voxel engine stuff on youtube (GabeRundlett, Ratty, Douglas, xima).

The main reason why I wanted to implement my own ray marcher was to implement this idea I've had for a wihle to use depth data from previous frames as an acceleration structure, which, as you read in the name of the blog post, deemed to be sufficient and robust enough. Ok enough yapping time to show how this shit worked

# *Why* I thought reprojection would work
In my head, I **knew** reprojection could be used to make ray-marching a lot quicker, since you could use the depth information from last frame and use *that* as a starting point for your starting position. Basically allows you to skip over all the early iterations since you know they won't hit anything. 

{% note(header="Note") %}
Now unfortunately this would only work for a static scene (or a scene where the depth of a pixel doesn't change over frames without any camera movement). For example, imagine a pixel having a depth $d$ at frame $t$. This ray-"advancing" optimization would only work if  $d_{t-1}>d_{t}$, since the pre-advanced ray would clip into the volume instead. Anyways.
{% end %}

I really *really* wanted to implement this since no-one else seems to have implemented it and I couldn't find anything about it online. This would also help me get my toes wet with reproj stuff in general since that's been a long standing goal of mine when it comes to shader stuff.

This is how I decided to implement it: split the reprojection stuff into two parts:
1) One part to handle rotational reprojection, since depth would not change over frames and since movement seems a bit too complex for me to handle directly.
2) Positional reprojection that takes account the new position of the camera. I originally feared that I'd need to do something like iterate within my reprojection step after I viewed this example of [VR reprojection](https://youtu.be/VvFyOFacljg?t=70) (which I had to do eventually but more on that later)

I started planning stuff out on paper thinking how this reprojection stuff based on the overly simplistic definition given by one of the members of a discord server I was in. 

*"nodle man" is me btw. Immense thanks to misha and rwighter in that server btw. Thankies for helpies :3*
![](/reprojection.png)
# Rotational Reprojection
Ok, so I tried implemented what ``misha`` suggested. Seems pretty simple right?
I could even simplify doing reprojection of the position by just reprojection the pixel ray *direction* instead, since that wouldn't change based on position but solely rotation!!! I am a genius!!!!



{% note(header="Note") %}
note text
{% end %}

## Heaps of fucking around leads to finding out
## Fucking stupidity V2 (the part where I lost my shit)

# Positional Reprojection

Okay-ish code that will calculate ray-direction based on uv space. Literally just boilerplate for any raymarching / raytracing stuff really. Do note that we must keep a ray direction vector **__NOT NORMALIZED__**. Very important 
```glsl
// apply projection transformations for ray 
vec4 ray_dir_non_normalized = inverse(proj_matrix) * vec4(coords, -1.0, 1.0);
ray_dir_non_normalized.w = 0.0;
ray_dir_non_normalized = inverse(view_matrix) * ray_dir_non_normalized;
vec3 ray_dir = normalize(ray_dir_non_normalized.xyz);
vec3 inv_dir = 1.0 / (ray_dir + vec3(0.0001));
```

Hideous code that actually handles calculating the uvs of the current pixel (more specifically pixel dir) in last frame uv-space
```glsl
vec4 last_uvs_full = (proj_matrix * last_frame_view_matrix) * vec4(ray_dir_non_normalized.xyz, 0.0);
last_uvs_full.xyz /= last_uvs_full.w;

// convert the [-1,1] range to [0,1] for texture sampling
vec2 last_uvs = last_uvs_full.xy;
last_uvs += 1;
last_uvs /= 2;
```

Positional reprojection loop. Basically ray-marching within the ray-marched depth of the previous / still frame. 

# Conclusion and results
# Final note
As a final note,