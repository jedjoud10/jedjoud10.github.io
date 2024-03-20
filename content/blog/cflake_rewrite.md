+++
title = "cFlake Engine Rewrite (maybe?)"
date = 2023-09-16
draft = true

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
# Goals:


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
2. No parallelisation between very small systems that read from shared resources or write to unique resources
3. No use of GPU for culling/instancing/multidraw. No bindless textures in use
4. Every system is fixed; cannot disable some and keep some active.
5. A lot of "drops" inside codes due to resource handling. Having to deal with internal mutability too.

**Good things with old cflake engine:**
1. Was simple to use and get running
2. Customizing material systems was pretty easy too
3. Parallel ECS system
4. Nice dependency injection using the events registry and staging system

**Goals with the rewrite:**
1. Handle parallelism with shred, should be extendable to material system without much trouble
2. Don't populate the code with "drop"s all over the place
3. Multithreadable so we can execute multiple systems at the same time
4. No race conditions
5. Not callable internally
6. Modular/customizable so we can disable specific systems or resources when not needed
7. Parallel GPU/CPU rendering using PHOBOS and 2 frames in flight
8. Better use of async maybe?
9. Somewhat unique to differentiate it from Bevy or any general 3D engine like Unity or Godot.
10. Use community driven crates instead of building everything from scratch (if not necessary).
11. Use Vulkan and SOLELY Vulkan (through Phobos) with build-time compiled shaders
12. Game editor possibly? Would simply be a different executable that will modify scenes and allow you to quickly execute your code.
14. Fast compile times (pls fix)
15. Doesn't run like absolute ass (should not be coping with sub 60 fps worst case)
16. Easy to maintain
17. Does not commit war crimes
18. Does not become a self ticking time bomb
19. Follows all ethical rules of the human race

**Bare bones stuff needed to get simple engine running:**
1. Event execution through custom dependency graph
2. Robust ECS system (multithreadable too)
3. Graphics API with the same old interface (type safe) BUT without abusing the type system and making everything a trait/generic (too slow for compilation)
4. Get EGUI running so we can implement a proper editor interface
5. Reflectable types so that we can make use of an editor interface