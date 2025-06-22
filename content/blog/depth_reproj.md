+++
title = "Ray-Marching and Temporal Depth Reprojection"
date = 2024-05-21
draft = false

[taxonomies]
categories = ["Compute"]
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
Now unfortunately this would only work for a static scene (or a scene where the depth of a pixel doesn't change over frames without any camera movement). For example, imagine a pixel having a depth $d$ at frame $t$. This ray-"advancing" optimization would only work if  $d_{t-1}>d_{t}$, since the pre-advanced ray would clip into the volume instead. 

Even with a small offset that you subtract from the approximated depth, you'd still run into issues if the depth changes without any camera movement (like modifying the geometry at runtime)
{% end %}

I really *really* wanted to implement this since no-one else seems to have implemented it and I couldn't find anything about it online. This would also help me get my toes wet with reproj stuff in general since that's been a long standing goal of mine when it comes to shader stuff.

This is how I decided to implement it: split the reprojection stuff into two parts:
1) One part to handle rotational reprojection, since depth would not change over frames and since movement seems a bit too complex for me to handle directly.
2) Positional reprojection that takes account the new position of the camera. I originally feared that I'd need to do something like iterate within my reprojection step after I viewed this example of [VR reprojection](https://youtu.be/VvFyOFacljg?t=70) (which is what I had to do eventually :( )

I started planning stuff out on paper thinking how this reprojection stuff based on the overly simplistic definition given by one of the members of a discord server I was in. 

"nodle man" is me btw. Immense thanks to ``misha`` and ``rwighter`` in that server btw. Thankies for helpies :3
![](/reprojection.png)
# Rotational Reprojection
Ok, so I tried implemented what ``misha`` suggested. Seems pretty simple right?
I could even simplify doing reprojection of the position by just reprojection the pixel ray *direction* instead, since that wouldn't change based on position but solely rotation!!! I am a genius!!!!

Unfortunately, this is where the majority of my problems began. Now, these problems weren't because I couldn't implement reprojection, but because my original ray-marching code (that I tested multiple times to make sure worked fine without repro) was flawed from the start. My math was wrong all along, and I was actually projecting *by* the viewproj matrix to calculate direction, which is absolutely **fucking** wrong!!!

What you actually need to do is take the inverse of the matrix and do vertex projection steps (like in a simple rasterized shader) but in the opposite order. I was very dumb and I spent like a solid 5 hours trying to figure out why my repro code didn't work ('cause in of itself it doesn't use that "inverted" viewproj matrix).

Anyways, after losing my mind over that problem I finally got some rotation reprojection working. Actually was a pretty robust solution and it worked nicely. There is the major flaw that this only worked with rotational reprojection, so any linear movement caused some rays to completely miss their targets, but for the intended purpose it worked really good.

# Positional Reprojection
Now is time for the relatively hard part; positional reprojection.
I had absolutely no idea how to do this because it isn't as straight forward as you think. When the camera moves from last frame's position to the current frame's position, there are multiple factors that could change how the scene depth changes during this translation.

If, for example, the camera moved forward in the direction it is currently looking at, the depth would decrease by a very slight amount that is proportional to the speed of the camera at that moment. This only holds true for the pixel at the very center of the screen (where uv = (0,0) if in range of [-1, 1]). For any other pixel the deph change is slightly less since we're using perspective projection and stuff.

As I had no idea what I was fucking doing, I looked up videos and other resources about this type of reprojeciton stuff, which led me to [Asynchronous Reprojection](https://en.wikipedia.org/wiki/Asynchronous_reprojection) for VR and this [demo view](https://youtu.be/VvFyOFacljg?t=70) implementing such a reprojection technique within the Unity game engine. As the wiki states, this technique is used in VR for cases where the hardware can't keep up with the software and causes the fps to drop, which is a very bad experience in VR, so this technique kinda uses the data from last frame and new rotation information from the headset to reproject the viewed image. This is basically what I want to do, but instead of reprojecting the whole image directly I would just repro the depth data to use in the current frame.

{% note(header="Note") %}
You could probably implement the async reprojection stuff on **top** of the current one to possibly speed it up even more, but at a latency cost kinda.
{% end %}

I decided to draw all of this on a piece of paper first to see how I would check the closest distance to the scene even after a "local" translation and rotation. This drawing just showcases two camera frustum, slightly offset from each other in one case and slightly rotated from each other in another case to see how I should remap the voxel geometry simply based on these differences. This actually helped me find out a flaw with my reprojection code which caused me some headaches cause I forgot to keep the ray direction un-normalized after I project it using the perspective matrix. 

*temporarily removed as I'm remaking it to make it clearer. Handwriting fucking sucks.*

Now, by the look of the demo video, it looked like some sort of iterative approach was at work because of the clear stepping that occured when the camera goes "inside" the old frustum and looking at it from the side. I kinda knew that it was doing something like ray-marching the old frustum itself, so that's what I tried to do first, and surprisingly that was literally how you implement positional reprojection (albeit probably the most naive method todo so)

# Actual code used
If you want to see the actual code behind this reprojection stuff here it is:
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
This was the code that specifically handled rotational reprojection in cases where the camera does not
```glsl
vec4 last_uvs_full = (proj_matrix * last_frame_view_matrix) * vec4(ray_dir_non_normalized.xyz, 0.0);

// the thing that I forgot that kinda made me insane
last_uvs_full.xyz /= last_uvs_full.w;

// convert the [-1,1] range to [0,1] for texture sampling
vec2 last_uvs = last_uvs_full.xy;
last_uvs += 1;
last_uvs /= 2;
```

Positional reprojection loop. Basically ray-marching within the ray-marched depth of the previous / still frame so we can keep a safe "minimum" distance to the scene even if we move any direction. 
Do note that before I fetch the depth texture of the last frame, I run a little downsampling / dilation step that simply blurs the depth texture and takes the smallest value in the convolution grid (which saves me to handle this in the main loop below).
I think you could even run the whole reprojection step at a lower resolution since the results are going to be dilated by some small amount anyways
```glsl
for (int i = 0; i < total_steps; i++) {
  vec3 temp_ray_dir = repr_pos - last_position;
  vec4 test_uvs = (proj_matrix * last_frame_view_matrix) * vec4(temp_ray_dir, 0.0);
  
  // the thing that I forot that kinda made me insane
  test_uvs.xyz /= test_uvs.w;
  vec2 test_uvs2 = test_uvs.xy;

  // convert the [-1,1] range to [0,1] for texture sampling
  test_uvs2 += 1;
  test_uvs2 /= 2;

  // check if the iterated position is within the old frustum
  if (test_uvs2.x > 0 && test_uvs2.y > 0 && test_uvs2.x < 1 && test_uvs2.y < 1 && test_uvs.w > 0) {
    float od = texture(last_temporal_depth, test_uvs2 / scale_factor).x;
    float nd = distance(last_position, repr_pos);

    // checking depth values
    if (od < nd) {
      pos_reprojected_depth = distance(position, repr_pos) - step_size * step_size_offset_factor;
      total_repro_steps_percent_taken = float(i) / float(total_steps);
      break;
    }
  }

  repr_pos += ray_dir * step_size;
}
```

# Conclusion and results
This optimization worked pretty nicely for most cases. There's definitely a big big room for improvement because I feel like the positional repro loop could be simplified using some sort of heuristic. Something like a HiZ depth pyramid could help us skip early steps. And for the rotational repo variant, it's really fast, but really really buggy whenever you move.
You can make it swap between rotational / positional reprojection based on camera movement, but it would be a bit erratic for the framerate to increase whenever you are not moving out of nowhere.

{% note(header="Idea") %}
If we treat each pixel on the screen as a mini frustum (with specific near/far bounds based on the last depth texture), we could do something like a line-frustum check to check the closest depth distance to scene that we can safely march by.
Basically getting rid of the whole inner frustum ray-marching step for the positional reprojector. No idea if this idea would work but would be interesting to experiment with this. 
{% end %}

{% note(header="Another Idea") %}
You could probably use subgroup operations and atomic operations to avoid marching through a lot in the volume by assuming that the minimum depth within a subgroup is close to the depth of each invocation / fragment within that subgroup. Cause if this is the case, then you could either do something like frustum/cone-marching instead of ray-marching for a coarse approximation and then do a finer *ray*-marched second step to get closer to the actual scene depth for each pixel in the subgroup.
{% end %}
