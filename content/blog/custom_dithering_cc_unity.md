+++
title = "Unity Pixelated Dithering & Color Compression in URP"
date = 2025-02-15
draft = false

[taxonomies]
categories = ["Unity"]
tags = ["unity", "graphics", "compute"]

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

Very short blog post on how I implemented a dithering and color compression effect in Unity using its new ``Render Graph API``.
The color compression part is implemented using a simple bit shifting (by getting rid of the first ``n`` bits of the r,g,b values). This is pretty simple and straightforward
The dithering however, was a bit more involved. I tried commenting what each part of the code does (at least in the shader part, the c# side is pretty straight forward).

```cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.RenderGraphModule;
using UnityEngine.Rendering.RenderGraphModule.Util;
using UnityEngine.Rendering.Universal;
public class DitherColorCompressionRenderFeature : ScriptableRendererFeature {
    class DitherColorCompressionPass : ScriptableRenderPass {
        private ComputeShader shader;
        private int compressionDepth = 7;
        private float multiplier = 128f;
        private float tightness = 0.0f;

        public DitherColorCompressionPass(ComputeShader shader, int compressionDepth, float multiplier, float tightness) {
            this.shader = shader;
            this.compressionDepth = compressionDepth;
            this.multiplier = multiplier;
            this.tightness = tightness;
        }

        private class PassData {
            public int width;
            public int height;
            public TextureHandle src;
            public ComputeShader shader;
        }
        public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData) {
            UniversalResourceData resourceData = frameData.Get<UniversalResourceData>();
            UniversalCameraData cameraData = frameData.Get<UniversalCameraData>();
            var desc = cameraData.cameraTargetDescriptor;
            desc.msaaSamples = 1;
            desc.depthBufferBits = 0;
            desc.enableRandomWrite = true;
            var tempTexture = UniversalRenderer.CreateRenderGraphTexture(renderGraph, desc, "TempTexture", false);

            Material blitterMat = Blitter.GetBlitMaterial(TextureDimension.Tex2D);
            renderGraph.AddBlitPass(new RenderGraphUtils.BlitMaterialParameters(resourceData.cameraColor, tempTexture, blitterMat, 0));

            using (var builder = renderGraph.AddComputePass<PassData>("Compute Dithering", out var passData)) {
                passData.src = tempTexture;
                passData.shader = shader;
                passData.width = desc.width;
                passData.height = desc.height;
                builder.UseTexture(passData.src, AccessFlags.ReadWrite);
                builder.AllowPassCulling(false);
                builder.SetRenderFunc((PassData data, ComputeGraphContext context) => {
                    context.cmd.SetComputeTextureParam(data.shader, 0, "screenTexture", data.src);
                    context.cmd.SetComputeIntParam(data.shader, "cDepth", compressionDepth);
                    context.cmd.SetComputeFloatParam(data.shader, "multiplier", multiplier);
                    context.cmd.SetComputeFloatParam(data.shader, "tightness", tightness);
                    context.cmd.DispatchCompute(data.shader, 0, Mathf.CeilToInt((float)data.width / 16.0f), Mathf.CeilToInt((float)data.height / 16.0f), 1);
                });
            }

            renderGraph.AddBlitPass(new RenderGraphUtils.BlitMaterialParameters(tempTexture, resourceData.cameraColor, blitterMat, 0));
        }
    }

    DitherColorCompressionPass pass;

    [Range(1, 8)]
    public int compressionDepth = 7;

    [Min(0)]
    public float multiplier = 128f;
    public float tightness = 0.0f;
    public ComputeShader customShader;

    public override void Create() {
        pass = new DitherColorCompressionPass(customShader, compressionDepth, multiplier, tightness);
        pass.renderPassEvent = RenderPassEvent.AfterRenderingPostProcessing;
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData) {
        if (renderingData.cameraData.cameraType == CameraType.Game) {
            renderer.EnqueuePass(pass);
        }
    }
}
```

```hlsl
#pragma kernel CSMain

// Properties!!
RWTexture2D<float4> screenTexture;
int cDepth;
float multiplier;
float tightness;

// Stolen from
// https://www.gamedev.net/articles/programming/general-and-gameplay-programming/inverse-lerp-a-super-useful-yet-often-overlooked-function-r5230/
float3 invlerp(float3 a, float3 b, float3 c) {
	return (c - a) / (b - a);
}

[numthreads(16,16,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
	// This NEEDS to be stored inside the function otherwise it won't work!!!
	// Thanks Unity!!!
	// Stolen from the unity URP shader graph dither node
	// https://docs.unity3d.com/Packages/com.unity.shadergraph@6.9/manual/Dither-Node.html
	const float DITHER_THRESHOLDS[16] = {
	    1.0 / 17.0,  9.0 / 17.0,  3.0 / 17.0, 11.0 / 17.0,
	    13.0 / 17.0,  5.0 / 17.0, 15.0 / 17.0,  7.0 / 17.0,
	    4.0 / 17.0, 12.0 / 17.0,  2.0 / 17.0, 10.0 / 17.0,
	    16.0 / 17.0,  8.0 / 17.0, 14.0 / 17.0,  6.0 / 17.0
	};

	// Dithering index
	uint index = id.x % 4 * 4 + id.y % 4;

	// Sample the screen textures at the appropriate coordinates
	float3 coloured = screenTexture[id.xy].xyz;

	// Convert the f32 rgb values to u8 (for bitshifting)
	uint3 converted = uint3(round(coloured * 255.0));

	// Do the bitshifting and covert back to float
	converted >>= (8 - cDepth);
	float3 occured = (float3(converted) / float(1 << cDepth));

	// Calculate some arbitrary "distance" between the compressed values and non-compressed values
	float3 dist = coloured - occured;

	// Some remapping logic? I think? Also actually samples the dither value
	float3 diffs = (dist * multiplier - 0.5) * 0.5 + 0.5;
	float3 temp = clamp(invlerp(tightness, 1.0, abs(diffs)), 0, 1);
	float3 dithering = temp * 2.0f - DITHER_THRESHOLDS[index];
	dithering = clamp(dithering, 0.0, 1.0);	

	// Kinda stupid to do this again but yea..
	uint3 rounded = uint3(round((coloured + dithering / 255.0) * 255.0));
	rounded = uint3(max(int3(rounded), 0));
	rounded = clamp(rounded, 0, 255);

	// Shift back and write to the texture!!
	rounded >>= (8 - cDepth);
	float3 tahini = float3(rounded) / float(1 << cDepth);
	screenTexture[id.xy] = float4(tahini, 0);
}
```