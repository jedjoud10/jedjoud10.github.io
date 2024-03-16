+++
title = "Per Vertex Ambient Occlusion for Surface Nets Meshes"
date = 2024-03-16
draft = false

[taxonomies]
categories = ["Mesh Generation"]
tags = ["procedural"]

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

In this short blog post I will show you how I implemented fake per-vertex ambient occlusion in both cFlake engine and my current Unity procedural terrain generator. First of all, here are the results of the ambient occlusion map in these two implementations.
For future reference ``AO`` refers to Ambient Occlusion. Too lazy to type it out :3

# Small Showcase
No AO
![Unity No AO](/unity_no_ao.png)

*With* AO. As you can see, the ambient occlusion adds quite a lot of depth to the terrain in low light situtations like in shadows. This effect is more pronounced in caves as well.
It affects a much larger area of the terrain compared to screen space effects like SSAO, and it is pretty cheap to compute as well.
![cFlake With AO](/unity_w_ao.png)


# Implementation
To implement such a system, you will first need a way to sample the density/isosurface function (the function I use to generate my procedural terrain using isosurface extraction algorithms) at any given point. This might be easy for some, but for me this was quite problematic as my procedural density generator is implemented on the GPU, and my voxel meshing is on the CPU. I will show you how I implemented it nonetheless)

## cFlake GLSL Implementation
This is the first time I experimented with this algorithm to even see if it would work out, and here's the code that I used to make it actually work. I calculate average ambient occlusio 

```glsl
// Get the density at a specific point (interpolated)
vec2 fetchLinear(vec3 position) {
    return texture(sampler3D(voxels, voxels_sampler), position / vec3(64.0)).xy;
}

// Calculate per vertex ambient occlusion by sampling close voxels and checking their densities
float occlusion(vec3 position, vec3 normal) {
    float ao = 0.0;

    for (int x = -1; x <= 1; x++) {
        for (int y = -1; y <= 1; y++) {
            for (int z = -1; z <= 1; z++) {
                ao += fetchLinear(vec3(position) + vec3(x, y, z) * 2.0 + vec3(1.0)).x > 0.0 ? 1.0 : 0.0;
            }
        }
    }

    ao = ao / (3*3*3);
    return ao;
}
```

As you can see, the algorithm in of itself is rather simple (and brute-forced). All I do is loop over a 3x3x3 region around the current vertex that I am about to generate and check how many density voxels are negative (marking terrain) vs how many density voxels are positive (marking air). Do note that you ``fetchLinear`` is a function implemented to get me the density function at any given floating point (not necessarily grid points) (trilinear sampling) using a per-chunk *cached* density map, which means that this algorithm does *not* read voxel data from nearby chunks which makes it easily parallelizable. 

AO debug view in cFlake
![cFlake AO Debug](/terrain_ao_pyramid.png)

For example this is how it looked like before implementing the ``fetchLinear``. You can clearly see the banding that occurs due to the fixed grid
![cFlake AO Banding](/banding.png)

## Unity Job System Implementation
The Unity implementation is not that different, just implemented on the CPU instead using Unity's Job System. I also made this one a bit prettier by applying a curve to the final AO and make it scale with the density of the fetched voxels, which reduces the *blocky*-ness effect that you get from using Surface Nets or Dual Contouring. 

```cs
// Calculate the normals at a specific position
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static float3 SampleGridNormal(uint3 position, ref NativeArray<Voxel> voxels, int size) {
    position = math.min(position, math.uint3(size - 2));

    float baseVal = voxels[PosToIndex(position)].density;
    float xVal = voxels[PosToIndex(position + math.uint3(1, 0, 0))].density;
    float yVal = voxels[PosToIndex(position + math.uint3(0, 1, 0))].density;
    float zVal = voxels[PosToIndex(position + math.uint3(0, 0, 1))].density;

    return new float3(baseVal - xVal, baseVal - yVal, baseVal - zVal);
}

// Calculate ambient occlusion around a specific point
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static float CalculateVertexAmbientOcclusion(float3 position, ref NativeArray<Voxel> voxels, int size, float offset, float power) {
    float ao = 0.0f;
    float minimum = 200000;
    
    for (int x = -1; x <= 1; x++) {
        for (int y = -1; y <= 1; y++) {
            for (int z = -1; z <= 1; z++) {
                float density = SampleGridInterpolated(position + new float3(x, y, z) * 2 + math.float3(0.9f), ref voxels, size);
                density = math.min(density, 0);
                ao += density;
                minimum = math.min(minimum, density);
            }
        }
    }

    ao = ao / (3 * 3 * 3 * (minimum + 0.001f));
    ao = math.clamp(1 - math.pow(ao + offset, power), 0, 1);
    return ao;
}
```

Here's a better picture of the AO in action in blocky unity terrain
![Unity AO](/unity_ao.png)

# Conclusion
Overall it's a pretty cheap algorithm that enhances the quality fidelty of the terrain without too much work. I personally consider that a success.
Here are other demos showcasing this effect

# Side Note
I did notice through my testing and experimentation with my engine that sometimes using the density function itself (or at least part of it) would yield some pretty cool results. Never managed to make use of it for AO but it looks *kinda* similar and cool enough so here you go. Maybe this could be extended to yield some new results, maybe. It was pretty cool though.

![cFlake Terrain Density Debug](/density.png)