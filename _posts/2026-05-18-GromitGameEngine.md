---
layout: post
author: Edward Burden
title : Gromit Game Engine
project :  C++
category : Updates
thumbnail: Gromit.png

excerpt: Starting my own game engine in C++.
---

Gromit is a game engine I began working on after playing around with the OpenGL tutorials.

There was a lot more I wanted to know about how to structure a game engine, and I wanted to have a go at starting one myself to find out where the difficult parts are. I loosely followed [The Cherno's game engine series](https://www.youtube.com/watch?v=JxIZbV_XjAs&list=PLlrATfBNZ98dC-V-N3m0Go4deliWHPFwT).

> **PLACEHOLDER:** Screenshot or short clip of Gromit running (the editor, a test scene, or the example application).

## Where I went my own way

I found the series a great resource, but from the beginning I wanted to make some differences. I wanted to expand on some of the systems that were only briefly set up in the series, mainly:

- **Event management**
- **Asset management**
- **Memory management**

> **PLACEHOLDER:** Short diagram or description of the overall engine structure (modules, and how they talk to each other).

## Data-driven scenes and prefabs

After getting model loading working in my last project, I wanted to find a proper way to describe data in text files that my engine could interpret. The goal was to make things like **scenes**, **prefabs** and **overrides** that could be loaded at runtime.

I ended up focusing more on these aspects than on rendering.

> **PLACEHOLDER:** Example of a scene file (the text format the engine reads).
>
> ```
> (paste scene file example here)
> ```

> **PLACEHOLDER:** Example of a prefab and an override of it.
>
> ```
> (paste prefab + override example here)
> ```

> **PLACEHOLDER:** Code snippet showing how the engine loads and interprets the file.

## Asset management

My plan for assets was to build a **packaging system**: bundle up the assets, compress them, and include them in builds. The engine would then be able to unzip or uncompress those bundles and read the assets back out of them. It is very similar to how Unity bundles up assets.

This is something I started on but never completed.

> **PLACEHOLDER:** Text or diagram of how the bundles were meant to work (the bundle format, compression, and how the engine would find an asset inside one). Code snippet of whatever I got built.
>
> ```cpp
> // paste asset manager / bundle example here
> ```

## Memory management

> **PLACEHOLDER:** Text about the memory management work (allocators, ownership, tracking). Code snippet or a short example.
>
> ```cpp
> // paste memory management example here
> ```

## Entity Component System

I managed to get a proper Entity Component System working. It wasn't the most efficient one, but I was happy with it, and it was a big step in understanding how engines structure their game objects.

> **PLACEHOLDER:** Text about how the ECS works (how entities, components and systems are stored and queried) and where it fell short on efficiency. Code snippet showing how you create an entity and add a component.
>
> ```cpp
> // paste ECS usage example here
> ```

## The event manager

This is a part I'm quite proud of. I didn't just follow what The Cherno did. I wanted using it to be as simple as this:

```cpp
// Register
REG_EVENT(Gromit::MouseClickEvent, this, DoSomethingMouse);

// Unregister
UREG_EVENT(Gromit::MouseClickEvent, this);
```

The idea is a really simple way for anywhere inside the engine, or in the user's application, to listen to events and immediately register a callback for when one is fired.

Here is an example of a callback in an application:

```cpp
bool ExampleApplication::DoSomethingWindow(Gromit::WindowsResizeEvent& event)
{
	GL_INFO("Width:{0}, Height:{1}", event.Width, event.Height);
	return false;
}
```

> **PLACEHOLDER:** Text or snippet explaining how the macros work underneath (what `REG_EVENT` expands to, how callbacks are stored, how unregistering works).
>
> ```cpp
> // paste the event manager implementation or macro definition here
> ```

## CMake

This was the first time I had configured a CMake project, and I found the process tough but rewarding. I still don't know everything about it, but I've used CMake in all of my projects since then, and I keep finding it super useful for setting up projects, switching configurations, and testing my work on different compilers and architectures.

It was also cool to be able to use CPack to turn my game into an installer, along with all of the assets the engine uses.

> **PLACEHOLDER:** Snippet of the `CMakeLists.txt` (project setup, or the CPack configuration).
>
> ```cmake
> # paste CMake example here
> ```

## Why I stopped

In the end, I found it very difficult to make a whole game engine in the time I have after work. I also found it hard to build out only little bits of the engine at a time, and I was more focused on making specific areas work, like asset loading or memory management.

A few things made it harder:

- **Size and complexity:** it was a huge project, and the scale of it overwhelmed me.
- **Still learning C++:** I didn't yet have a great grasp of the language, and I kept worrying that I wasn't making the right decisions about the architecture.
- **No game to build:** I didn't have a clear game in mind for the engine, so I wasn't making decisions to create a product.

So I stopped work on the engine. The idea now is to build these parts in isolation, and when I'm more confident in my abilities, put them back together. There are definitely parts of the project that I'm proud of, and I'll keep using what I learned from it.

> **PLACEHOLDER:** Link to the GitHub repository, and anything else worth adding (lessons learned, what I'd do differently next time).