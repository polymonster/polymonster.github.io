---
title: 'Archive'
subtitle: 'Older projects from 2004-2013'
date: 2020-09-24 00:00:00
featured_image: '/images/archive/deferred-rendering.gif'
---

A collection of older projects from 2004-2013, some work was completed during my time at university and others were side projects. They helped with my development as a programmer and I learned a lot in the process. I am keeping them around for posterity, the source code and executables (Windows only) are available on [GitHub](https://github.com/polymonster/demos-archive).

### Radiosity (2010)

An implementation of radiosity rendering algorithm, light sources are rendered emissively and other surfaces recieve incoming light by rendering hemi-cube maps per texel on the scenes surfaces from unwrapped UV's. The hemi-cubes are downsampled and average to give per texel incoming radiance this process is repeated multiple times to achieve multiple light bounces.

<img src="/images/archive/radiosity.gif">

### Deferred Rendering / Post-Processing (2009)

Implemented with OpenGL and C++ this was around the time when deferred rendering and light pre-pass rendering techniques started to emerge on games consoles and I was working on Xbox360/PS3 titles. It features a deferred renderer with point lights, bloom, depth of field, HDR and per pixel motion blur post-processing effects.

<img src="/images/archive/deferred-rendering.gif">

### Antechamber (2009)

A pixel art 2D platformer made by me and a friend as part of a university project. It was written in C++ using Direct3D9c, we made a tile-map level editor which allowed for slopes and walls, different enemies, obstacles and game mechanics, audio and multi-player support. This was, at the time of completion, the most complete and polished project I had worked on and we received 92% mark for the course work.

<img src="/images/archive/antechamber.gif">

### Organic Editor (2009)

My final year university project, I created a level editor with terrain painting / deformation, model loading, object manipulation, and various shader techniques. It was written in C++ using OpenGL and GLSL shaders.

<img src="/images/archive/editor.png">

### Sandwick Arena (2008)

A 3D Arena style game with third-person camera. I worked on this over the summer in between my 3rd year (placement year) and final year of university. It was was written in C++ with OpenGL using the fixed function pipeline, I implemented multi-player support via local area networking using sockets over TCP.

<img src="/images/archive/sandwickarena.png">
