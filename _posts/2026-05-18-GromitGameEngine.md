---
layout: post
author: Edward Burden
title : Gromit Game Engine
project :  C++
category : Updates
thumbnail: Gromit.png
github: https://github.com/EdwardBurden/Gromit
excerpt: Starting my own game engine in C++.
---

<style>
.alloc-code { background: rgba(127, 127, 127, 0.12); color: inherit; padding: 1rem; border-radius: 0.5rem; overflow-x: auto; font-size: 0.875rem; }
.alloc-code code { color: inherit; background: none; }
mark.hl { color: inherit; padding: 0.05rem 0.2rem; border-radius: 0.2rem; }
mark.hl-1 { background: rgba(0, 123, 255, 0.30); }
mark.hl-2 { background: rgba(40, 167, 69, 0.32); }
mark.hl-3 { background: rgba(255, 153, 0, 0.38); }
mark.hl-4 { background: rgba(156, 39, 176, 0.30); }
</style>

Gromit is a game engine I began working on after playing around with the OpenGL tutorials.

There was a lot more I wanted to know about how to structure a game engine, and I wanted to have a go at starting one myself to find out where the difficult parts are. I loosely followed [The Cherno's game engine series](https://www.youtube.com/watch?v=JxIZbV_XjAs&list=PLlrATfBNZ98dC-V-N3m0Go4deliWHPFwT).

I found the series a great resource, but from the beginning I wanted to make some differences. I wanted to expand on some of the systems that were only briefly set up in the series, and on the things I had more of an interest in. Those are the things I'd like to highlight below. The project is available on GitHub, but it's a lot of code and not all of it is polished, so I think it's best to focus on these parts.

## Scene and asset loading

After getting model loading working in my last project, I wanted to find a proper way to describe data in text files that my engine could interpret. The goal was to be able to set up assets that depend on each other, and load the whole chain of requirements. A texture and a shader are used by a material, materials and meshes are used by a prefab, and prefabs are used in scenes. The idea is to load them into and out of memory, and send them to the GPU when needed.

Here's an example of the scene loaded when the engine starts. I used JSON, mainly because it's something I'm familiar with, and I found it easy to interpret and to change by hand.

```json
{
  "guid": 5,
  "type": "Scene",
  "name": "Test_Scene_1",
  "path": "",
  "entities": [
    {
      "instance": 47647,
      "guid": 1,
      "overrides": [
        {
          "componentID": 0,
          "componentType": "Transform",
          "position": {
            "x": 3,
            "y": 0,
            "z": -10
          }
        },
        {
          "componentID": 1,
          "componentType": "MeshRenderer",
          "material": 999
        }
      ]
    },
    {
      "instance": 5553,
      "guid": 1,
      "overrides": [
        {
          "componentID": 0,
          "componentType": "Transform",
          "position": {
            "x": -3,
            "y": 0,
            "z": -10
          }
        }
      ]
    }
  ]
}
```

The scene loads two prefabs, both with the same GUID. It then applies overrides to the transform and mesh renderer components, so that they are moved away from the root of the scene, and one of them gets a material showing a smiling face.

This is what it looks like when loading and unloading this scene:

<img class="img-fluid rounded shadow mb-4" src="/assets/images/gromit_load.gif" alt="Gromit loading and unloading the test scene">

## Asset handlers

One of the things I wanted to achieve with my asset management setup was being able to create new scripts for loading new types of assets without having to amend anything in my asset manager class. I did this by creating a new handler per asset type. Let's look at textures to see how it works.

```cpp
struct TextureMetaData : public MetaData
{
public:
    int m_width;
    int m_height;
    int m_channels;
    TextureFormat m_format;
    bool m_generateMipMaps;

    TextureMetaData(int guid, std::string assetPath, nlohmann::json data);// : MetaData(guid, assetPath, data);
};

struct TextureAsset : public Asset
{
    int m_width;
    int m_height;
    int m_nrChannels;
    unsigned char* m_data; // todo improve

    TextureAsset(std::shared_ptr<TextureMetaData> metaData);
    ~TextureAsset();
};
```

Each asset defined in JSON translates to two classes: the metadata, and the actual asset. This is what a texture looks like in JSON:

```json
{
  "guid": 345455675675,
  "type": "Texture",
  "name": "Smile_Texture",
  "path": "/smile.png",
  "width": 256,
  "height": 256,
  "format": "RGB",
  "generateMipMaps": true
}
```

And this is the actual asset it describes:

<img class="img-fluid rounded shadow mb-4" src="/assets/images/smile.png" alt="The smile texture used by the Smile_Texture asset">

The actual asset might have a large memory footprint, but the metadata is small. That means I can load an asset's metadata and store it in a metadata class, then load the actual asset later if it's required. Both are stored in an `AssetRecord`.

Some assets directly load a file that they require. In the future this might need to be multiple files, but for now it's limited to one. A texture is made up of the metadata the engine needs to use it, plus the actual texture file. Other assets, like scenes and entities, are made up of just metadata.

### Registering a handler

In the texture asset's `.cpp` file you'll see this macro:

```cpp
REGISTER_ASSET_HANDLER(TextureAssetHandler, AssetType::Texture, TextureAsset, TextureMetaData, 8);
```

It is defined like this:

```cpp
#define REGISTER_ASSET_HANDLER(HANDLER_CLASS, TYPE, ASSET, META, POOL_SIZE) \
class HANDLER_CLASS : public AssetHandler<ASSET, META, POOL_SIZE> \
{ \
public: \
    HANDLER_CLASS() { AssetDatabase::RegisterHandler(TYPE, this); } \
}; \
static HANDLER_CLASS s_##HANDLER_CLASS
```

For each asset class there is a handler, and the handler is registered with the asset database. The asset database manages all of the handlers, and decides which one to use based on the metadata it reads when it opens an asset's JSON file.

The handler registers itself at runtime. The macro creates a static instance of the handler, and its constructor calls `RegisterHandler`. That is what lets me add new asset types and have them loaded, with no switch statements or changes to the asset manager.

## Loading a scene

Loading starts from preloaded metadata about the scene and its dependencies. This is done at the start of the program, when a bundle of assets is identified:

```cpp
Gromit::ServiceLocator::GetService<Gromit::AssetManager>()->IdentifyAssets("assets");
```

That gives us the metadata half of every asset up front, and the actual assets can be loaded later, as they're needed. From there, the loading goes like this:

<div class="my-4">
<svg viewBox="0 0 640 444" width="100%" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Loading flow: identify assets at start up, load the scene, create each entity, create each component, then load their dependencies" style="max-width: 640px; display: block; margin: 0 auto; font-family: inherit;">
<defs><marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="currentColor"/></marker></defs>
<rect x="60" y="16" width="520" height="60" rx="8" fill="rgba(0,123,255,0.16)" stroke="rgba(127,127,127,0.55)" stroke-width="1.5"/>
<text x="76" y="40" font-size="15" font-weight="600" fill="currentColor">1. Start up<tspan font-weight="400" font-family="monospace" font-size="13" opacity="0.8">   IdentifyAssets("assets")</tspan></text>
<text x="76" y="62" font-size="13" fill="currentColor" opacity="0.75">Loads metadata for every asset, scene and dependencies included</text>
<line x1="320" y1="76" x2="320" y2="101" stroke="currentColor" stroke-width="1.5" opacity="0.7" marker-end="url(#arrow)"/>
<rect x="60" y="104" width="520" height="60" rx="8" fill="rgba(127,127,127,0.12)" stroke="rgba(127,127,127,0.55)" stroke-width="1.5"/>
<text x="76" y="128" font-size="15" font-weight="600" fill="currentColor">2. Load the scene<tspan font-weight="400" font-family="monospace" font-size="13" opacity="0.8">   LoadScene(guid)</tspan></text>
<text x="76" y="150" font-size="13" fill="currentColor" opacity="0.75">Loads the scene asset: the list of entity references</text>
<line x1="320" y1="164" x2="320" y2="189" stroke="currentColor" stroke-width="1.5" opacity="0.7" marker-end="url(#arrow)"/>
<rect x="60" y="192" width="520" height="60" rx="8" fill="rgba(127,127,127,0.12)" stroke="rgba(127,127,127,0.55)" stroke-width="1.5"/>
<text x="76" y="216" font-size="15" font-weight="600" fill="currentColor">3. Create each entity<tspan font-weight="400" font-family="monospace" font-size="13" opacity="0.8">   EntityInstance::Create</tspan></text>
<text x="76" y="238" font-size="13" fill="currentColor" opacity="0.75">Loads the entity asset, then goes through its components</text>
<line x1="320" y1="252" x2="320" y2="277" stroke="currentColor" stroke-width="1.5" opacity="0.7" marker-end="url(#arrow)"/>
<rect x="60" y="280" width="520" height="60" rx="8" fill="rgba(127,127,127,0.12)" stroke="rgba(127,127,127,0.55)" stroke-width="1.5"/>
<text x="76" y="304" font-size="15" font-weight="600" fill="currentColor">4. Create each component<tspan font-weight="400" font-family="monospace" font-size="13" opacity="0.8">   ComponentInstance</tspan></text>
<text x="76" y="326" font-size="13" fill="currentColor" opacity="0.75">Loads the assets that the component needs</text>
<line x1="320" y1="340" x2="320" y2="365" stroke="currentColor" stroke-width="1.5" opacity="0.7" marker-end="url(#arrow)"/>
<rect x="60" y="368" width="520" height="60" rx="8" fill="rgba(127,127,127,0.12)" stroke="rgba(127,127,127,0.55)" stroke-width="1.5"/>
<text x="76" y="392" font-size="15" font-weight="600" fill="currentColor">5. Dependencies<tspan font-weight="400" font-family="monospace" font-size="13" opacity="0.8">   Meshes and materials</tspan></text>
<text x="76" y="414" font-size="13" fill="currentColor" opacity="0.75">Along with the textures and shaders they use</text>
</svg>
</div>

Here's how a scene is loaded:

<pre class="alloc-code"><code>void SceneManager::LoadScene(int guid)
{
    <mark class="hl hl-1">auto sceneAsset = ServiceLocator::GetService&lt;AssetManager&gt;()-&gt;LoadAsset&lt;SceneAsset&gt;(guid);</mark>
    m_activeScene = new SceneInstance();
    m_activeScene-&gt;Entities.clear();
    <mark class="hl hl-2">for (auto&amp; entity : sceneAsset-&gt;GetEntities())</mark>
    {
        <mark class="hl hl-2">const auto&amp; entityInstance = EntityInstance::Create(entity);</mark>
        m_activeScene-&gt;Entities.push_back(entityInstance);
    }
}</code></pre>

It starts by <mark class="hl hl-1">finding and loading the scene asset</mark>, which is just the JSON that tells us what entities are in the scene. Later on there could be scene specific information, like lighting, but for now it's just entities and their overridden components.

<mark class="hl hl-2"><code>GetEntities</code> returns a list of entity references</mark>, and <mark class="hl hl-2">each one is used to create an entity</mark>:

<pre class="alloc-code"><code>EntityInstance* EntityInstance::Create(const EntityReference&amp; entityRef)
{
    const auto&amp; assetManager = ServiceLocator::GetService&lt;AssetManager&gt;();
    EntityInstance* entity = new EntityInstance();
    entity-&gt;GuiID = entityRef.Guid;
    entity-&gt;InstanceID = entityRef.Instance;
    entity-&gt;Components.clear();
    <mark class="hl hl-1">entity-&gt;AssetRef = assetManager-&gt;LoadAsset&lt;EntityAsset&gt;(entityRef.Guid);</mark>
    <mark class="hl hl-3">for (auto&amp; [key, component] : entity-&gt;AssetRef-&gt;GetComponents())</mark>
    {
        <mark class="hl hl-3">ComponentType type = component.get()-&gt;GetComponentType();</mark>
        <mark class="hl hl-3">switch (type)</mark>
        {
        <mark class="hl hl-3">case ComponentType::Transform:</mark>
        {
            <mark class="hl hl-4">auto ptr = ComponentInstance::CreateComponentInstance&lt;TransformComponentInstance&gt;(key, component, entityRef);</mark>
            entity-&gt;Components.emplace(type, std::move(ptr));
            break;
        }
        <mark class="hl hl-3">case ComponentType::MeshRenderer:</mark>
        {
            <mark class="hl hl-4">auto ptr = ComponentInstance::CreateComponentInstance&lt;MeshRendererComponentInstance&gt;(key, component, entityRef);</mark>
            entity-&gt;Components.emplace(type, std::move(ptr));
            break;
        }
        default:
            break;
        }
    }
    return entity;
}</code></pre>

Like the scene, <mark class="hl hl-1">the entity is its own asset, which is loaded using its GUID</mark>. It holds the entity's components, so the function <mark class="hl hl-3">loops through them and, depending on the component type, creates the matching one</mark>. For now that is the transform and the mesh renderer.

For each one, <mark class="hl hl-4">a <code>ComponentInstance</code> is created</mark>, which will eventually load the assets that component requires.

Here is what that looks like for the mesh renderer. `SetComponent` sets the instance up from its component asset: <mark class="hl hl-1">it loads the mesh and material assets</mark> that the asset refers to, and <mark class="hl hl-2">copies across the component's ID and type</mark>.

The overrides from the scene are then applied with `ApplyProperty`. <mark class="hl hl-3">If the property's name is <code>mesh</code> or <code>material</code>, and its value is an integer</mark> (the GUID of an asset), <mark class="hl hl-4">the current asset is reset and the new one is loaded in its place</mark>. This is how the override in the scene JSON, `"material": 999`, gives one of the two entities its smiling face material.

<pre class="alloc-code"><code>void MeshRendererComponentInstance::SetComponent(const std::unique_ptr&lt;ComponentAsset&gt;&amp; compAsset)
{
    const auto&amp; meshAsset = static_cast&lt;MeshRendererAsset*&gt;(compAsset.get());
    const auto&amp; assetManager = ServiceLocator::GetService&lt;AssetManager&gt;();
    <mark class="hl hl-1">mesh = assetManager-&gt;LoadAsset&lt;MeshAsset&gt;(meshAsset-&gt;m_mesh);</mark>
    <mark class="hl hl-1">material = assetManager-&gt;LoadAsset&lt;MaterialAsset&gt;(meshAsset-&gt;m_material);</mark>
    <mark class="hl hl-2">ComponentID = meshAsset-&gt;GetID();</mark>
    <mark class="hl hl-2">ComponentType = meshAsset-&gt;GetComponentType();</mark>
}

void MeshRendererComponentInstance::ApplyProperty(const std::string&amp; name, const ComponentProperty&amp; value)
{
    const auto&amp; assetManager = ServiceLocator::GetService&lt;AssetManager&gt;();
    <mark class="hl hl-3">if (name == "mesh" &amp;&amp; std::holds_alternative&lt;int&gt;(value))</mark>
    {
        <mark class="hl hl-4">mesh.reset();</mark>
        <mark class="hl hl-4">mesh = assetManager-&gt;LoadAsset&lt;MeshAsset&gt;(std::get&lt;int&gt;(value));</mark>
        return;
    }

    <mark class="hl hl-3">if (name == "material" &amp;&amp; std::holds_alternative&lt;int&gt;(value))</mark>
    {
        <mark class="hl hl-4">material.reset();</mark>
        <mark class="hl hl-4">material = assetManager-&gt;LoadAsset&lt;MaterialAsset&gt;(std::get&lt;int&gt;(value));</mark>
        return;
    }
}</code></pre>

## Looking back

This system is complicated. It was my first attempt, and while I tried to keep the code segmented, object oriented and clear, it has led to a lot of obfuscation. If I were redoing it, I would start with something a lot more compact and easy to change.

Overall I'm glad I had a go at it, and it does work, but because of the size of the project it has become difficult to improve. I'd like to remake it as a separate project, which my engine could then include as a submodule.