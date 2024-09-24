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
```cs
public abstract class SerializableProp : MonoBehaviour, INetworkSerializable {
    // Set the prop as modified, forcing us to serialize it
    public bool wasModified = false;

    // Dispatch group ID bitmask index used for loading and saving 
    public int ElementIndex { internal set; get; } = -1;

    // Stride of the byte data we will be writing
    public abstract int Stride { get; }

    // Variant of the prop type
    public int Variant { get; internal set; }

    // Called when the fake gameobject for capturing gets spawned
    public virtual void OnSpawnCaptureFake(Camera camnera, Texture2DArray[] renderedTextures, int variant) { }

    // Called when the fake gameobject for capturing gets destroyed
    public virtual void OnDestroyCaptureFake() { }

    // When a new prop spawns :3
    public virtual void OnPropSpawn(BlittableProp prop) { }

    // Serialization / deserialization on a per prop basis
    public abstract void NetworkSerialize<T>(BufferSerializer<T> serializer) where T : IReaderWriter;
}
```

## C# Capture Code
And finally, this is the code that handles creating the textures and capturing the data using the custom shader right afterwards.
```cs
Texture2DArray albedoTextureOut = new Texture2DArray(width, height, prop.variants.Count, TextureFormat.ARGB32, mips);
Texture2DArray normalTextureOut = new Texture2DArray(width, height, prop.variants.Count, TextureFormat.ARGB32, mips);
Texture2DArray maskTextureOut = new Texture2DArray(width, height, prop.variants.Count, TextureFormat.ARGB32, mips);
Texture2DArray[] tempOut = new Texture2DArray[3] { albedoTextureOut, normalTextureOut, maskTextureOut };

// Many different variants per prop type
for (int i = 0; i < prop.variants.Count; i++) {
    PropType.PropVariantType variant = prop.variants[i];
    camera.orthographicSize = variant.billboardCaptureCameraScale;

    // Spawns the fake object and put it in its own layer with the camera
    GameObject faker = Instantiate(variant.prefab);
    faker.GetComponent<SerializableProp>().OnSpawnCaptureFake(camera, tempOut, i);
    faker.layer = 31;
    foreach (Transform item in faker.transform) {
        item.gameObject.layer = 31;
    }

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

    // Reset the prop variant
    faker.GetComponent<SerializableProp>().OnDestroyCaptureFake();
    faker.SetActive(false);
    Destroy(faker);
    temp.DiscardContents(true, true);
    temp.Release();
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
# Part 2: Rendering the impostorss