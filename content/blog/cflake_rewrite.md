+++
title = "cFlake Engine Rewrite"
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

If I ever pickup working on cFlake engine again, I will definitely need to rewrite most if not all of it. Not for the sake of going lower level and doing things manually, butfor the sake of using community driven libraries and imrpoving compile times to improve prototyping speeds (for actual use in game development). I think that was one of the most exhausting things I've had to deal with when writing the test examples for ``cFlake`` engine, the process was simply too tedious and the time lost due to slow Rust compilation was very high.

So, in my next rewrite, I'll have to focus on actually building a reliable engine instead of trying to flex and build everything from scratch (*which is imo a mega-flex but a mega waste of time as well*). 