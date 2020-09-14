---
title: 'pmtech'
subtitle: 'Lightweight, multi-platform, data-oriented game engine.'
date: 2020-09-13 00:00:00
featured_image: '/images/pmtech/gifs/post-pro.gif'
---

pmtech is a data-oriented game engine written in c++, it is open source and available on [github](https://github.com/polymonster/pmtech). The engine is cross platform and supports windows, macOS, iOS, linux and wasm/webgl with android under development. It also has multiple rendering backends with Direct3D11, OpenGL, Metal and Vulkan.

I have been using it as a framework to experiment and try different rendering techniques, it has a data-oriented component entity system and features such as live reloading of c++ code, shaders and render pipeline configs to enable rapid prototyping and real time development. There is a live video demo of the hot reloading capabilities on [youtube](https://youtu.be/dSLwP4D8Fd4).

Below are some of the more advanced and interesting graphics techniques I have implemented, there are over [40 examples](https://github.com/polymonster/pmtech/wiki/Examples) and unit tests in the repository to aid testing and cross platform compatibility.

### Realtime Global Illumination
Realtime voxel cone traced GI with colour shadow map voxel volumes.

[<img src="/images/pmtech/gifs/gi.gif" width="1280" />](https://github.com/polymonster/pmtech/blob/master/examples/code/area_lights/area_lights.cpp)

### Area Lights
Linearly transformed cosines, animated ray marched primitives, mip-mapped roughness.

[<img src="/images/pmtech/gifs/area-lights.gif" width="1280" />](https://github.com/polymonster/pmtech/blob/master/examples/code/area_lights/area_lights.cpp)

### Subsurface Scattering
Sepearable sss configured with [pmfx](https://github.com/polymonster/pmtech/wiki/Pmfx) and [ecs](https://github.com/polymonster/pmtech/wiki/Ecs).

[<img src="/images/pmtech/gifs/sss.gif" width="1280" />](https://github.com/polymonster/pmtech/blob/master/examples/code/sss/sss.cpp)

### Signed Distance Field Shadows 
Precalculated 3D signed distance field generated in pmtech editor, ray marched in real time.

[<img src="/images/pmtech/gifs/sdf-shadow.gif" width="1280" />](https://www.youtube.com/watch?v=369cPinAhdo)

### Data Driven Renderer
Config driven renderer using jsn config files, 100 lights, switch between deferred, fwd and zprepass.

[![Renderer](/images/pmtech/gifs/pmfx-renderer.gif)](https://github.com/polymonster/pmtech/blob/master/examples/code/pmfx_renderer/pmfx_renderer_demo.cpp)

### Data Driven Post Processing
Config driven post processing using jsn configs, ray marched menger sponges, bloom, dof, crt, colour correction.

[![Post Processing](/images/pmtech/gifs/post-pro.gif)](https://github.com/polymonster/pmtech/blob/master/examples/code/post_processing/post_processing.cpp)

### Stencil Shadow Volumes
Multi-pass lighting with depth pass stencil shadow volumes, easily configured with [ecs](https://github.com/polymonster/pmtech/wiki/Ecs) and [pmfx](https://github.com/polymonster/pmtech/wiki/Pmfx).

[<img src="/images/pmtech/gifs/stencil-shadows.gif" width="1280" />](https://github.com/polymonster/pmtech/blob/master/examples/code/stencil_shadows/stencil_shadows.cpp)

### Entities
64k entities into 4 shadow maps at 60hz, showing the strengths of data-oriented [ecs](https://github.com/polymonster/pmtech/wiki/Ecs).

[<img src="/images/pmtech/gifs/tourus.gif" width="1280" />](https://github.com/polymonster/pmtech/blob/master/examples/code/entities/entities.cpp)

### Vertex Stream Out / PBR
Vertex stream out, transform feedback meshes rendered many times, cook-torrence + oren-nayar BRDF.

[![Vertex Stream Out](/images/pmtech/gifs/vertex-stream-pbr.gif)](https://github.com/polymonster/pmtech/blob/master/examples/code/vertex_stream_out/vertex_stream_out.cpp)
