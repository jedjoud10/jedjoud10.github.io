+++
title = "The state of Unity ECS in 2025"
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
- Umm.... it's.... it's le ECS.... hehehehehe

# Cons
## Pit Holes
- ``ISystem`` and ``SystemBase`` ``OnCreate`` executes even before some baked entities get created in the world. Meaning that if you want to fetch a baked singleton component in one of your sub-scenes you need to do something like *this*:
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
- ``ISystem`` can NEVER use or even REFERENCE a managed class type. Even if you don't have ``[BurstCompile]``, you still can't use it in ``ISystem``
- Having a baked child (with a non (1,1,1) scale and with a collider component) will not bake it as a child. Any transformations done on the parent entity will not be propagated to that child or its children. No warning or anything like that to let you know either... lol
- Due to code gen, using generics (or generic query functions) becomes a bit harder. Still doable as long as you don't use ``SystemAPI``
- There is no way to force two colliders (a child and a parent) to not be merged during baking when using ``Unity Physics``. So far, I have not found a solution for this. This is really annoying.

# SystemAPI discrepencies 
- There are missing ``EntityManager`` methods in ``SystemAPI``.
- There does not exist a ``SystemAPI.TryGetComponent`` nor ``SystemAPI.TryGetComponentRW``. You have to do a manual ``HasComponent`` check and then fetch the component data
- There does not exist a ``EntityManager.GetComponentDataRW`` for entities, like ``SystemAPI.GetComponentRW`` for entities.

# Entity Baking / Conversion Workflow
- The ```Entity Baking System```. It's painful that all the GameObject prefabs that can be spawned at runtime must be first converted to Baked Entities. This stops you from using ``Scriptable Objects`` to store ``GameObject`` prefabs to instantiate, as you must keep the data in an ``Authoring`` ``MonoBehaviour`` with corresponding ``Baker``. For example, the following data-driven code:
```cs
public class SwordData: ScriptableObject { 
    public float hitDamage;
    public float aoeDamage;
    public float range;
    public GameObject hitPrefab;
}
```

must be converted to all of THIS to be able to instantiate ``hitPrefab`` as an entity
```cs
public class SwordDataAuthoring: MonoBehaviour { 
    public float hitDamage;
    public float aoeDamage;
    public float range;
    public GameObject hitPrefab;
}

public class SwordDataBaker: Baker<SwordDataAuthoring> {
    public override void Bake(SwordDataAuthoring authoring) {
        Entity self = GetEntity(TransformUsageFlags.None);

        AddComponent(self, new SwordData {
            hitDamage = authoring.hitDamage,
            aoeDamage = authoring.aoeDamage,
            range = authoring.range,
            hitPrefab = GetEntity(authoring.hitPrefab, TransformUsageFlags.Dynamic)
        });
    }
}
```

{% note(header="Edit") %}
Because you must first add it to the entities baking dependency chain for conversion. That whole system should be removed and instead of using gameobjects, Unity should have implemented an Entity only workflow. Adding that level of complexity (converting from GameObjects to Entities) complicates things, as you can only do that at runtime.
{% end %}

I mean, I understand why they'd opt for this. Having all the data being serialized in entities before spawning them makes loading scenes with lots of entities really quick, as it's just a memory copy practically. No expensive loading. My issue is how they have built the entity conversion workflow on *top* of GameObject based MonoBehaviours instead of coming up with a dedicated solution specifically for ECS. 

Unity states that "you should be able to mix and match ECS and GameObjects in your projects" but that practically never happens. Either because you need one ECS-specific feature in ECS that isn't supported by MonoBehaviours or because you need one GameObject specific feature that isn't supported by ECS. It's half baked, basically. If they made the runtime Entity component editor able to create entity "Prefabs" and set their components that way, that could save us some trouble, but I bet it wouldn't be easy considering the other design choices.


- ``MeshRenderer``s with multiple materials create multiple additional entities during baking, but there's no way for the user to know about these additional entites. This screws you over if you want to keep track of all the mesh renderers of a converted ``GameObject`` so that you could, for example, change their ``RenderFilterSettings`` value at runtime (for example, for changing the light layer value at runtime). At least Unity converts the multi-mat ``MeshRenderer``s to Parents in the ECS world, so you could iterate over its children, but that's still somewhat stupid.

# Blob Data
- No way to check or debug blob data in the editor. Looking at a component with a ``BlobAssetReference`` will NOT show the internal blob data that was baked. There isn't a blob data window tab either. It's as if blobs do not even exist in the editor lol.

{% note(header="Lol, lmao even") %}
\> https://blog.s-schoener.com/2024-12-17-unity-blobs/ <br> 
\> "Blob assets in C# were an experiment, and it failed."
{% end %}