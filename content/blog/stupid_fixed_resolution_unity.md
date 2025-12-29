+++
title = "Unity Fixed Resolution Render Feature"
date = 2025-12-28
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

In Unity URP, When you try to render to a ``RenderTexture``, the camera will use the global `Univeral Render Pipeline Asset`'s ``render scale`` parameter. Even though you are writing to a ``RenderTexture`` with a fixed resolution.
This is problematic and causes blurring in the texture after it has been written to. 

This is a simple render feature and pass that alleviates that by forcing the camera to render at a given resolution, no matter the ``render scale``.
This is very stupid.

```cs
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.RenderGraphModule;
using UnityEngine.Rendering.Universal;

public class FixedResolutionFeature : ScriptableRendererFeature {
    // set these to any value you want
    // you can even make them public parameters that you can modify in the editor
    public const int WIDTH = 128;
    public const int HEIGHT = 128;


    public class FixedResolutionPass : ScriptableRenderPass {
        public FixedResolutionPass() {
            renderPassEvent = RenderPassEvent.BeforeRenderingOpaques;
        }

        public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData) {
            UniversalResourceData resourceData = frameData.Get<UniversalResourceData>();
            UniversalCameraData cameraData = frameData.Get<UniversalCameraData>();
            var desc = cameraData.cameraTargetDescriptor;
            desc.width = WIDTH;
            desc.height = HEIGHT;
            desc.useDynamicScale = false;
            desc.useDynamicScaleExplicit = true;
            cameraData.cameraTargetDescriptor = desc;
        }
    }

    FixedResolutionPass pass;

    public override void Create() {
        pass = new FixedResolutionPass();
    }

    public override void AddRenderPasses(
        ScriptableRenderer renderer,
        ref RenderingData renderingData) {
        renderingData.cameraData.cameraTargetDescriptor.width = WIDTH;
        renderingData.cameraData.cameraTargetDescriptor.height = HEIGHT;
        renderer.EnqueuePass(pass);
    }
}
```