+++
title = "New programming project ideas"
date = 2024-04-27
draft = false

[taxonomies]
categories = ["Random"]
tags = []

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

Here I will keep track of new project ideas that I eventually want to experiment with. Simply writing this down cause most of the time when I try to go code I ask myself "what should I work on this time??" and then never find anything to work on only even though I've had many thoughts about the things I wanted to work on not even 20 minutes before that.
Ok so here's a list of random projects I've had an idea to work on but I neither didn't have the time / will to work on them.

# Voxel renderer
Just a simple voxel game engine with infinite terrain generation. I do want to try to implement either ray-marching / ray-tracing / 3D DDA to be able to view a shit-load of voxels without having the need for an octree or LOD around the player as that complicates things quite a bit. I also want to experiment with hard / soft ray-traced voxel shadow. In theory sounds really cool cause I could experiment with rust-gpu and coding most of the tracing / GPU code in rust which would be nice as it would allow me to use the trait system and stuff like that. Maybe using phobos or raw vulkan instead of wgpu as well?? Would be cool to experiment with.

{% note(header="Edit") %}
I am currently in the process of writing an OpenGL / OpenTK ray-marcher. Will eventually post the repo online with some cool demo videos. So far I've implemented a few optimizations that I am currently writing a [__blog post__](http://jedjoud10.github.io/blog/depth-reproj/) about actually lol
{% end %}

# Fully GPU sided renderer
Just a renderer with basic features like PBR rendering and other shiz (like in cFlake engine) but instead of having the CPU contain all the data you would instead store literally *everything* on the GPU. Textures, buffers, and everything else would be generated/handled by the GPU (could maybe implement some sort of procedural texture generation as well). But yea culling and everything else would be handled on the GPU, which could lead to some very nice performance boosts compared to the renderer in cFlake for example.

If I'm ever going to do such a thing (which I'm getting hyped up to do now) I really want to go out with the "GPU" side shenanigans. If my GPU supports it, I wanna try out buffer device addresses and other new extension for bindless stuffs. Here's a list of the relatively "new" stuff that I haven't messed around with and that I really REALLY want to mess around with:
* Bindless rendering
* Buffer Device Address
* Ray tracing
* Atomics, Subgroup & Warp operations 

# SDF / Ray-marching editor hierarchy viewer
Just a simple test with SDF shapes and egui to make something similar to Womp but that runs locally instead.

# cFlake Engine 2
Look at the "cFlake Engine Rewrite (maybe?)" blog to see all what's wrong with the current engine and what I wanna work on for cFlake 2

# Genetic neural network thingy
Basically takes the basic of a simple neural network (input -> output with weights/biases within them & hidden layers) but instead of using back propagation we spawn a ton of random entities with random biases & biases and keep the ones that do well alive and then "breed" it (just means spawning new entities with the traits with the old ones but with slight mutations, and then just keep doing it)
Could be even accelerated with GPU!!! I do want to see how this would like over multiple generations, maybe using a compute shader or something.
First tests kinda work for evaluating the color of a ball given a camera feed of the ball (yellow, green, blue, red) albeit the hidden neurons are just absolutely random lol. 

# Custom system dispatcher
Custom system dispatcher like shred but instead of implementing traits and using associated types we do everything at runtime. So like the user would have to specify before-hand what resources they would read/write from/to and then use that in the frame. Sounds cool to implement at least. Would allow me to experiment with different heuristics for re-ordering systems and other stuffs at runtime. Managed to scrap the old dipsatching / injection system of cflake and build this using petgraph. ~~Currently the repo is hidden but it's mostly used for experimentation at the moment.~~ I made the repo public owo~~
Here it is: [bleghh](https://github.com/jedjoud10/dispatcher-system)
It was very fun to toy around with, albeit the "merging" algorithm that makes stuff run in parallel isn't the best at the moment. It definitely needs a rework or two to be actually usable (especially with how bad the code and commenting is)

# WASM4 voxel renderer
Pretty self-explanatory. Just a voxel renderer but in wasm4 instead.

# WASM4 inverse FFT midi auto-song thingy
Basically this would take an audio file for a song and try to run an inverse fourier transform on it to find bass/ main melody of a song to be then stored to be played afterwards using the dedicated noise, triangle, and sine wave channels in the wasm4 fantasy emulator. 

{% alert(header="BIG FUCKING WARNING") %}
I am sorry in advance for the crimes I have committed on the face of God's green Earth.
{% end %}

# Brainrot-rs
Custom scripting language with an interpreter written in Rust. Just full on brainrot stuff. I currently have a working implementation that could parse something like this:
```brainrot
let me be fr rn
let you be inting rn
chat, is this inting?
let me be inting rn
let you be inting rn
nuhuh
eep for 10 billions
yap
a
mew
ls in the chat
is you fr
```

# Factory game but with DAG
Factory game but instead of having factories be real entities in the world they're all nodes part of a big DAG (directed acyclic graph). This could allow you to have a **lot** of entities without sacrificing performance. Could allow you also write resource generation rate as a mathematical formula instead of having actual items/resources be moved from point A to point B. What you could also do with this implementation is have "buffer" nodes that literally act as buffer that would act as temporary places to store resources (really just the formula) and then swapping it out later.

# Potator (potato 2D game in space)
Currently working on this in Godot engine as a way to learn the engine. Just like old potator but this time it won't run like shit like in Unity. Currently have implemented pistol weapon and frier weapon (shotgun). Also added some cool dash ability and planet sprouts that will allow you to get more ammo.

# Third person anime fighting combo game
Just had the idea of merging an anime fighting game just like kurtzpel. So you could have some sort of whole combo system with semi-precise frame timings. 
I'm so fucking angry how the kurtzpel devs just shat on the game for the last few years. It used to be such a good game with a somewhat interesting (albeit repetive) story on the PvE side, but now it's just absolute dog shit effort at a money grabbing scam. It had such good mechanics too, and I constantly look at the steam page and they keep adding characters with their custom abilities which sucks cause the game went to fucking shit.
I already tried Genshin Impact, Wuthering Waves, and Blue Protocol. None of them
1) ran nicely on my laptop
2) gave me the same feeling / type of fun that kurtzpel did

I am set on making this game unless something similar to Kurtzpel gets released. No idea how I'm going to handle making the art style but I really want to make this game a thing.

# Karnaugh map solver using a neural network
Using the previous neural network coding idea maybe I could implement a karnaugh map solver that could take any inputs / outputs and reduce them to their simplest boolean formula using a karnaugh map. 

# Electrilized
This was actually the very first game I wanted to implement when I started having a passion for game development. Basically just a factory game with procedurally generated terrain and somewhat accurate electronic components (like capacitors, resistors, stuff like that). Originally implemented in UE4 on my shitty Toshiba laptop, but I haven't tried reimplementing the project for over 6 years now. It originally had low poly style with a grid like map and an undeground dungeon system.
Logan brought up the idea of making this a liminal space game instead. Not what I originally intended with the OG game but sounds fun so might as well. 

# AI based discord bot
Already implemented this in JavaScript with a hacky API I found online to fetch character AI (c.ai) responses from a user sent message. Most of the time the shit broke down somewhere in the API call pipeline but I haven't fixed that yet. 
This was cool as it allowed me to talk to imaginary anime girls which was pretty funny.

{% note(header="Update!") %}
Started working on this again! Now doesn't crash as much (only when the input suggests something violent / explicit) and it works pretty nicely for making me feel loved by virtual people.
{% end %}

# Something with OpenTK
OpenTK is pretty fun to mess with as well. I wanna either implement a game engine or an actual game within it.

{% note(header="Note") %}
Actually managed to do something with OpenTK!!!! It's the voxel ray-marcher thingy I mentioned above...
{% end %}

# Game with procedural magic circle / magic system
Idea I've had for a while actually. Fully blown out scalable and modular magic system system that allows you to create spells, runes, and incantations that follow some underlying laws (like laws of equivalent exchange / conservation of energy. stuff that makes sense).
Would be cool to make a procedural spell system that allows you to use up nearby resources (mana in the air, mana from user, materials around you) and modify it or use it as an input resource for other sub-spells. Could lead to a "pipeline" effect where multiple spells depend on each other's outputs, kinda like a factory. 

All from some simple elements: Light, Water, Stone, Organic, Mana, Ether
Could for example, make a spell that uses stone particles and converts it to other types using mana
And each spell / rune would be describable using the patterns within it, like a special pattern could represent "using" an element as input or "outputting" it to the next part of the pipeline


<center>Example of how it could kinda work?</center>
{% mermaid() %}
%%{ init: { 'flowchart': { 'curve': 'stepBefore' } } }%%
flowchart LR
A[Mana from user]
A --> B[Use up material from surrounding]
B --> C[Transmute using another material]
C --> H[Materialize into physical object]

B --> D[Mold into shape]

D --> E[Boost as projectile]
E --> G[Pellet, Shape, Dust]

{% end %}
 