+++
title = "cFlake Engine Rewrite (maybe?)"
date = 2023-09-16
draft = false

[taxonomies]
categories = ["cFlake Engine"]
tags = ["game-dev", "ecs", "game-engine-dev"]

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
# Internal Libraries: 
* app: default initialization point. Should allow customizing global settings and registering events and such
* assets: Solid and stable asset management system. Should be multithreadable as well, and should allow for recursive asset loading (for gltf files for example). Could be used for loading scenes.
* audio: Good audio support (with optional hrtf) and real-time raytraced audio (optional). Should allow playback, recording, generating, and mixing / synthesis
* graphics: Vulkan based Graphics API, wrapped with proper results and Validation layer when running in debug mode. (probably going to use a fork of [phobos](https://crates.io/crates/phobos))
* gui: [yakui](https://crates.io/crates/yakui) with graphics API backend
* input: old system was ok, probably should make the new one call events instead of having to constantly check for inputs
* utils: simple utilities
* networking: idk yet
* physics: idk yet (rapier was okay but definitely could be better)
* rendering: simple rendering based on graphics crate
* world management: next topic

**Problems with old cflake engine:**
1. Not GPU parallel safe. Rendering would occur at the end of the frame, causing us to wait for the GPU to complete work. No latency, but bad impact on performance.
    * Could be fixed by making a copy of the scene or the needed data (like object matrices) at the start of the frame and IMMEDIATELY beginning rendering (would add latency)
2. No parallelisation between very small systems that read from shared resources or write to unique resources
3. No use of GPU for culling/instancing/multidraw. No bindless textures in use
4. Every system is fixed; cannot disable some and keep some active.
5. A lot of "drops" inside codes due to resource handling. Having to deal with internal mutability too.
6. CPU overhead due to not having a GPU-sided renderer. Mostly taken up by the resource fetch overhead too

**Good things with old cflake engine:**
1. Was simple to use and get running
2. Customizing material systems was pretty easy too
3. Parallel ECS system
4. Nice dependency injection using the events registry and staging system
5. Cool to toy with (especially messing around the rendering engine)

**Goals with the rewrite:**
* Handle parallelism with shred (or custom dispatcher), should be extendable to material system without much trouble
* Don't populate the code with "drop"s all over the place
* Multithreadable so we can execute multiple systems at the same time
* No race conditions
* Parallel GPU/CPU rendering using 2-3 frames in flight
* Game editor possibly? Would simply be a different executable that will modify scenes and allow you to quickly execute your code.
    * On second hand not really. As much as I love phobos in theory it's pretty unstable rn (panics without a good reason) even though the underlying code makes intuitive sense :(.
* Better use of async maybe?
* Somewhat unique to differentiate it from Bevy or any general 3D engine like Unity or Godot.
* Use community driven crates instead of building everything from scratch (if not necessary).
* Use Vulkan and SOLELY Vulkan (through Phobos) with build-time compiled shaders
* Fast compile times (pls fix)
* Doesn't run like absolute ass (should not be coping with sub 60 fps worst case)
* Easy to maintain
* Does not commit war crimes
* Does not become a self ticking time bomb
* Follows all ethical rules of the human race

**Bare bones stuff needed to get simple engine running:**
1. Event execution through custom dependency graph
2. Robust game framework system (multithreadable too)
    * So not necessarily an ECS, which would be tricky to do in Rust (due to borrowing stuff)
3. Graphics API with the same old interface (type safe) BUT without abusing the type system and making everything a trait/generic (too slow for compilation)
4. Get EGUI running so we can implement a proper editor interface
5. Reflectable types so that we can make use of an editor interface