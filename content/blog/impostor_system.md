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

In this short blog post I'll demonstrate how I was able to draw multiple thousand impostors and their variants, just like in the following figures:
{{ figure(src="/500k_trees.png", caption="Half a million impostors (counting unculled)") }}

{{ figure(src="/775k_total.png", caption="") }}
{{ figure(src="/775k_total_counts.png", caption="340k visible impostors") }}

{{ figure(src="/1mil_standalone.png", caption="") }}
{{ figure(src="/1mil_count.png", caption="1 million visible impostors at 40fps@1440p on a 780m (massively bottlenecked due to sub-pixel triangles / warp underutilization)") }}

These impostors support diffuse maps, normal maps, mask maps (roughness, metallic, ao), and directional light shadows! (through a shader graph little hack I'll dwell on a bit later)
My system uses an **"auto-capture"** method that automatically captures all the necessary textures for the impostors of each prop type at the start of the game, which makes it so that I don't have to manually create these textures myself which improves development time. 

This post will be slightly? more in detail on technical jargon so here's just a list of words that I used and over and their definitions
* Prop: General terrain "object". Just a game object that is spawned during chunk generation that will then be converted to an impostor at further distances
* Impostor: Basically, billboard on drugs, uses a simple billboard but uses pre-captured/pre-calculated textures that give the billboard some "depth"
* Diffuse Texture: Contains only the color data of something to be rendered
* World Normal Texture: Contains the world normals of something to be rendered. Different than normal maps since those store their data in tangent space, whilst this stores it in world space. I use these for my impostors as it personally makes it easier to sample in the shader, instead of having to do some extra stuff that I didn't bother with.
* Mask Texture: Contains extra data like metallic, AO, and smoothness all compressed into the R,G,A channels of the texture (in that order specifically)

## Big stuffs
This system can be split up to 3 **main** parts:
1. Capturing required texture data during initialization for the required mesh variants
2. Creating buffers that contain the transform and variant types for each impostor type. 
3. Rendering the impostors using a custom shader and using Unity graphics commands directly.

{% note(header="Note") %}
In my terrain package, creating the compute buffers for impostor data and filling them up is done automatically. 
However, I will explain what steps my program takes whenever that occurs
{% end %}

# Part 1: Capturing required data
This is the most interesting and most important step of the system, as this is where we will decide the main factors that affect how our impostors will look like:
* How the required render textures are captured
* What parameters we should use to capture them (camera rotation, distance, scale)

For my current implementetion, I've settled for only using albedo, normal, and mask (roughness, metallic) data from my real meshes. Since I am using the Unity game engine, I have written all the required parameters and required meshes down into multiple scriptable objects that I feed to my capture script at the staart of the game.
Here's a screenshot of what impostor data you could give to it:

![](/prop_data.png)

So the main things to keep in mind here are the following data entries:
* Variants: A list of multiple prefabs and billboard capture settings (camera transform and distance)
* Prop spawn behavior: How props should handle rendering
* Billboard capture settings:
    * Texture width & height: **EXTREMELY** important! This depicts the size of the textures that would be used for billboard rendering
    * Filter mode and mipmap mode: Self explanatory. Sometimes I found mipmaps to be too blurry so I added this setting to just disable them completely
* GPU Instance Rendering: Very useful as well since this allows us to enable or disable shadows for the impostors (using a little shadertoy hack I found out that I will explain later). Also allows us to lock the rotation in the Y direction. Very useful for trees as you wouldn't want them to really *look* at the camera, as pop-in would become really visible when you look at it from above / below.

I capture the diffuse / world normal / mask data using a custom shader that I apply to the spawned prop mesh temporarily before destroying the mesh. 
Since we can have multiple variants per prop, I decided to use ``Texture2DArray`` instead of multiple textures since that would allow me to sample multiple indices of that texture array within the shader directly.
The props all contain the following script to customize this bevahiour:

## C# Prop Code
Using this class, I can handle prop serialization (which is another system in of itself, might write a post about how I handle that too), but most importantly, allows me to override the ``OnSpawnCaptureFake()`` method which is called right before we "capture" the render texture information for that prop (and its variants if needed). 
```cs,filename
public abstract class SerializableProp : MonoBehaviour, INetworkSerializable {
    // ...

    // Called when the fake gameobject for capturing gets spawned
    public virtual void OnSpawnCaptureFake(Camera camnera, Texture2DArray[] renderedTextures, int variant) { }

    // ...
}
```

## C# Capture Code
And finally, this is the code that handles creating the textures and capturing the data using the custom shader right afterwards.
```cs
// Creates required texture arrays that store each type of texture
Texture2DArray albedoTextureOut = new Texture2DArray(width, height, prop.variants.Count, TextureFormat.ARGB32, mips);
Texture2DArray normalTextureOut = new Texture2DArray(width, height, prop.variants.Count, TextureFormat.ARGB32, mips);
Texture2DArray maskTextureOut = new Texture2DArray(width, height, prop.variants.Count, TextureFormat.ARGB32, mips);
Texture2DArray[] tempOut = new Texture2DArray[3] { albedoTextureOut, normalTextureOut, maskTextureOut };

// Many different variants per prop type
for (int i = 0; i < prop.variants.Count; i++) {
    PropType.PropVariantType variant = prop.variants[i];
    camera.orthographicSize = variant.billboardCaptureCameraScale;

    // Spawns the fake object and put it in its own layer with the camera
    // ...

    // Move the prop to the appropriate position
    faker.transform.position = variant.billboardCapturePosition;
    faker.transform.eulerAngles = variant.billboardCaptureRotation;

    // Renders the camera 3 times (albedo, normal, mask)
    for (int j = 0; j < 3; j++) {
        tempOut[j].filterMode = prop.billboardTextureFilterMode;
        propCaptureFullscreenMaterial.SetInteger("_TextureType", j);
        camera.Render();
        for (int m = 0; m < temp.mipmapCount; m++) {
            Graphics.CopyTexture(temp, 0, m, tempOut[j], i, m);
        }
    }

    // Release resources and destroy fake capture object
    // ...
}
```

## Shader
The unity shader code that handles outputting a passthrogh fullscreen unshaded image based on the required texture type (either diffuse, normals, mask) 
```glsl
float3 color = float3(0, 0, 0);

NormalData normalData;
DecodeFromNormalBuffer(posInput.positionSS.xy, normalData);

if (_TextureType == 0) {
    color = LOAD_TEXTURE2D_X(_GBufferTexture0, posInput.positionSS).rgb;
}
else if (_TextureType == 1) {
    color = normalData.normalWS.xyz;
    color = (color + 1) / 2.0;
} else {
    // contains roughness, metallic (HOW), maybe occlusion?
    color = float3(1 - normalData.perceptualRoughness, 0, 0);
}

float alpha = LOAD_TEXTURE2D_X(_GBufferTexture0, posInput.positionSS).a;
```

# Part 2: Creating prop graphics buffers
This is another fun part of the process, which is creating the buffers that contain the packed data for each prop. I have this C# struct that contains all relevant data that is then sent to the GPU to be unpacked whenever we try to render the props. 
```cs
// Blittable prop definition (that is also copied on the GPU compute shader)
[StructLayout(LayoutKind.Sequential)]
public struct BlittableProp {
    // Size in bytes of the blittable prop
    public const int size = 16;

    public half pos_x;
    public half pos_y;
    public half pos_z;
    public half scale;

    // 3 bytes for rotation (x,y,z)
    public byte rot_x;
    public byte rot_y;
    public byte rot_z;
    
    // 1 unused padding byte
    public byte _padding;

    // 2 bytes for dispatch index
    public ushort dispatchIndex;

    // 1 byte for prop variant
    public byte variant;

    // 1 unused padding bytes
    public byte _padding1;
}
```

Most of the fields in this struct are using half precision or packed precision types to make impostor rendering take as little memory as possible. Unfortunately this also comes at the cost of having a lower precision readback for whenever we actually want to spawn the props as gameobjects, but we'll just have to cope with that for now.

{% note(header="Note") %}
There's a way to fix this and it's just to have a separate "Readback" buffer containing another blittle prop struct type with higher precision fields that's only used for gameobject spawning. Could work out maybe... (right now I haven't had a single precision issue with my system so I'll just keep it as is)
{% end %}

Anyways, I have the same structure also stored on the GPU so that I can create a big fat ``RWStructuredBuffer``s for all the props. For performance reason, I don't have multiple buffers for each type of prop, and instead I just pre-allocate a big *big* fat one and hope that the capacity values that the user specified in the scriptable object are to be trusted (otherwise we'd be writing past the capacity of the buffer lels) 

Here's the code that generates those buffers for example:
```cs
// Initialize CPU and GPU buffers
private void InitGpuRelatedStuff() {
    // Temp buffers used for first step in prop generation
    int tempSum = props.Select(x => x.maxPropsPerSegment).Sum();
    tempCountBuffer = new ComputeBuffer(props.Count, sizeof(int), ComputeBufferType.Raw);
    tempPropBuffer = new ComputeBuffer(tempSum, BlittableProp.size, ComputeBufferType.Structured);

    // Secondary buffers used for temp -> perm data copy
    tempIndexBuffer = new ComputeBuffer(props.Count, sizeof(int), ComputeBufferType.Raw);
    int permSum = props.Select(x => x.maxPropsInTotal).Sum();
    int permMax = props.Select(x => x.maxPropsInTotal).Max();
    maxPermPropCount = permMax;
    permPropBuffer = new ComputeBuffer(permSum, BlittableProp.size, ComputeBufferType.Structured);
    permBitmaskBuffer = new ComputeBuffer(permMax, sizeof(uint), ComputeBufferType.Structured);

    // Tertiary buffers used for culling
    culledCountBuffer = new ComputeBuffer(props.Count, sizeof(int));
    drawArgsBuffer = new GraphicsBuffer(GraphicsBuffer.Target.IndirectArguments, props.Count, GraphicsBuffer.IndirectDrawIndexedArgs.size);
    int visibleSum = props.Select(x => x.maxVisibleProps).Sum();
    culledPropBuffer = new ComputeBuffer(visibleSum, BlittableProp.size);

    // More settings!!!
    maxDistances = new float[props.Count];
    meshIndexCount = new uint[props.Count];
    maxDistanceBuffer = new ComputeBuffer(props.Count, sizeof(float));
    meshIndexCountBuffer = new ComputeBuffer(props.Count, sizeof(uint));

    // Other stuff (still related to prop gen and GPU alloc)
    propSectionOffsetsBuffer = new ComputeBuffer(props.Count, sizeof(int) * 3);
    propSectionOffsets = new uint3[props.Count];
    segmentIndexCountBuffer = new ComputeBuffer(VoxelUtils.MaxSegments * props.Count, sizeof(uint) * 2, ComputeBufferType.Structured);
    propSegmentDensityVoxels = VoxelUtils.Create3DRenderTexture(VoxelUtils.PropSegmentResolution, GraphicsFormat.R32_SFloat);
    unusedSegmentLookupIndices = new NativeBitArray(VoxelUtils.MaxSegments, Allocator.Persistent);
    unusedSegmentLookupIndices.Clear();
    segmentsToRemoveBuffer = new ComputeBuffer(VoxelUtils.MaxSegmentsToRemove, sizeof(int));
}
```

I just initialize all the buffers at the start with the maximum capacity that they will ever hold. This cuts down on program complexity as well since I don't have to keep track of the currently allocated length to "re-allocate" the memory 

# Part 2: Rendering the impostors
## Compute Based Dot Product (stupid) Culling
## Shader graph hack for instanced indirect