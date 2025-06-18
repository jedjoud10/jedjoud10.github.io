+++
title = "The State & Usability of Unity ECS in 2025"
date = 2025-05-21
draft = true

[taxonomies]
categories = ["Unity"]
tags = ["unity", "ecs"]

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

Up until recently, I have chosen not to mess with Unity ECS due to its sheer complexity (and often talked about *over*-complexity compared to something like ``flecs``) and poor documentation / poor usage in the community.
This changed a few days ago when we decided to start writing a new game using ECS to try to get a feel for the technology and see if using it in our other projects could deem doable and worthwhile.
Here is the list of the nuances / pitholes, and pros that we feel should be more discussed about before doing what we did (heading completely blind without looking anything up before hand)

# Pros
- Easy to create "entity" like behaviour without worrying about what should be inherited and what should be composited; everything *will* be composited


# Cons

## Pit Holes
- ``ISystem`` and ``SystemBase`` ``OnCreate`` executes even before some baked entities get created in the world. Meaning that if you want to fetch a baked singleton component to spawn something only **once** you need to do something like *this*:
```cs
public partial struct System: ISystem {
    private bool initialized;

    public void OnCreate(ref SystemState state) {
        initialized = false;
    }

    public void OnUpdate(ref SystemState state) {
        if (!initialized && SystemAPI.TryGetSingleton<MySingleton>(out MySingleton singleton)) {
            initialized = true;

            // do wtv you want here... will be called ONCE 
        }
    }
}
``` 
like what the **fuck**? why can't I just add a ``[CreateAfter(typeof(SomeSceneLoadingSystem))]`` instead and call it a day?????

- Why is it that some functions require you to use ``SystemState`` whilst others require you to use ``SystemAPI``?? Why can't you keep it consistent...
- ``ISystem`` can NEVER use or even REFERENCE a managed class type. Even if you don't have ``[BurstCompile]``, you still can't use it in ``ISystem``
- Due to code gen, using generics (or generic query functions) becomes a bit harder. Still doable as long as you don't use ``SystemAPI``