+++
title = "Reprojecting depth values as an acceleration structure for ray-marching"
date = 2024-05-21
draft = true

[taxonomies]
categories = ["c#", "openTK", "openGL", "glsl", "graphics-dev"]
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

In this short blog post, I will show you how I managed to use reprojected last frame depth information as an optimization for my C#/OpenGL voxel ray-marcher. 