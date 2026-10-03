---
layout: post
author: Edward Burden
title : Learning OpenGL
project :  C++
category : Updates
thumbnail: learningOpenGl.png
github: https://github.com/EdwardBurden/Learning_OpenGL

excerpt: Learning how to render things in C++ with OpenGL.
---

As part of relearning C++, I've been working through OpenGL, since rendering is one of the main areas I want to understand properly. After years of Unity doing all of this for me, it was interesting to build it up from the ground up.

I followed the excellent [LearnOpenGL](https://learnopengl.com/) site from start to finish.
<div class="row mb-4">
<div class="col-md-4 mb-3 mb-md-0">
<img class="img-fluid rounded shadow" src="/assets/images/learningOpenGl.png" alt="OpenGL scene screenshot 1">
</div>
<div class="col-md-4 mb-3 mb-md-0">
<img class="img-fluid rounded shadow" src="/assets/images/LearningNormal.png" alt="OpenGL scene with normal mapping">
</div>
<div class="col-md-4">
<img class="img-fluid rounded shadow" src="/assets/images/learening depth.png" alt="OpenGL scene screenshot 3">
</div>
</div>

## What I covered

- **Window creation:** setting up an OpenGL context and getting a window on screen.
- **Shaders:** writing vertex and fragment shaders and seeing how data flows through the graphics pipeline.
- **Textures:** loading images and mapping them onto geometry.
- **Lighting:** working with light sources and how they affect the look of a surface.
- **Shadows:** adding shadows to the scene.
- **Loading models:** importing 3D models into the renderer.

## Making my own model

To test that I understood the model loading, I made my own model with a texture and imported it into the project. It loaded successfully, which was a satisfying moment, because it meant the whole pipeline worked with my own content, not only the tutorial's assets.

## What's next

Seeing the whole pipeline come together has made a lot of what Unity does for me feel much less like magic. Next I want to keep building on this and apply it to my other C++ projects.