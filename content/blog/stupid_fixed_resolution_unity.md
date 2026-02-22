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


{% note(header="2026-01-07 Edit") %}
I have modified the render feature and pass to be able to automatically fetch the render texture width and height *from* the camera directly.
This is much neater to use, and could be used for multiple cameras that are rendering to differently sized render textures.
{% end %}

```cs
public class FixedResolutionFeature : ScriptableRendererFeature {
    public class FixedResolutionPass : ScriptableRenderPass {
        public FixedResolutionPass() {
            renderPassEvent = RenderPassEvent.BeforeRenderingOpaques;
        }

        public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData) {
            UniversalResourceData resourceData = frameData.Get<UniversalResourceData>();
            UniversalCameraData cameraData = frameData.Get<UniversalCameraData>();
            var desc = cameraData.cameraTargetDescriptor;
            var target = cameraData.camera.targetTexture;
            desc.width = target.width;
            desc.height = target.height;
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

        RenderTexture target = renderingData.cameraData.camera.targetTexture;

        if (target != null) {
            renderingData.cameraData.cameraTargetDescriptor.width = target.width;
            renderingData.cameraData.cameraTargetDescriptor.height = target.height;


            BuildingHologramRenderer hologramRenderer = Object.FindFirstObjectByType<BuildingHologramRenderer>();

            if (hologramRenderer == null)
                return;

            renderer.EnqueuePass(pass);
        } else {
            Debug.LogWarning("FixedResolutionFeature only works with cameras that have a render texture target");
        }

    }
}

```