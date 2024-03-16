+++
title = "Per Vertex Ambient Occlusion for Surface Nets Meshes"
date = 2024-03-01
draft = true

[taxonomies]
categories = ["Mesh Generation"]
tags = ["procedural"]

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

In this short blog post I will show you how I implemented fake per-vertex ambient occlusion in both cFlake engine and my current Unity procedural terrain generator. First of all, here are the results of the ambient occlusion map in these two implementations.

![cFlake AO Debug](/terrain_ao_pyramid.png)
![cFlake Normal View](/terrain_ao_pyramid_noao.png)


![Unity No AO](/unity_no_ao.png)
![cFlake With AO](/unity_w_ao.png)
