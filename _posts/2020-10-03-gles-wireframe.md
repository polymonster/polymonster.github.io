---
title: 'A quick solution for OpenGLES lack of wireframe fill mode'
date: 2020-09-24 00:00:00
---

If you have ever worked with OpenGL you might be familiar with [`glPolygonMode`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glPolygonMode.xhtml). It allows you to specify `GL_LINE`, `GL_POINT` or `GL_FILL`, where fill is typically what we use to rasterise solid triangles, using lines allows us to achieve a wireframe effect. If you have ever then switched to OpenGLES (mobile or web platforms) you may have encountered `GL_LINE` being undefined and there is no way to get a wireframe fill mode.

This typically isn't too much of a problem because wireframe rendering is quite a debug feature, or it could be used for some kind of stylised rendering... it just isn't that important really. It only started to affect me in one small case when I ported my engine [pmtech](https://github.com/polymonster/pmtech) to WebGL and one of the samples `physics_constraints` started exhibiting some ghastly z-fighting (ewww):

![broken](/images/posts/wireframe/constraints-wireframe-broken.gif)

The sample draws some lines to show constraint hinges and points as well as showing physics bounding volumes in wireframe. In the sample the wireframe is not that important, but it is useful in the context of debug rendering physics bounding volumes that are often an approximation of underlying geometry. Having the wireframe allows you to see the geometry underneath and assess how efficient the bounding volume is. On other platforms (Direct3D11, Metal, Vulkan, OpenGL 3.0+) this was working OK. When I switched to WebGL I just omitted the code regarding `glPolygonMode` to get everything compiling and to begin with I had much bigger problems to deal with than this minor cosmetic issue.

I considered removing the need for wireframe entirely by just creating some debug primitives by using line lists, but this raised some issues with my debug rendering API that stores a monolithic buffer of line lists and has no way to transform the vertices per instance by an object world matrix. I could transform each vertex by a world matrix before pushing it into the buffer, but this would take CPU cycles to perform a per vertex matrix multiply that I would rather do on the GPU... After a little bit of thought I came up with a solution that is not perfect and has some edge cases but is good enough for my use.

Having a custom graphics API [abstraction layer](https://github.com/polymonster/pmtech/blob/master/core/pen/include/renderer.h) offers many benefits. I have been able to make Metal and Vulkan rendering backends behave like a c-style Direct3D11 front end. With this abstraction you can do all kinds of gymnastics to make different API's behave the same as each other and give yourself high level platform agnostic code which can effortlessly target multiple platforms.

To make OpenGLES allow wireframe style draw calls the trick is to make any draw calls made with `GL_TRIANGLES` to actually draw with `GL_LINE_STRIP`. I can easily intercept this with my concept of rasteriser state:

```c++
struct rasteriser_state_creation_params
{
    u32 fill_mode = PEN_FILL_SOLID;
    u32 cull_mode = PEN_CULL_BACK;
    s32 front_ccw = 0;
    s32 depth_bias = 0;
    f32 depth_bias_clamp = 0.0f;
    f32 sloped_scale_depth_bias = 0.0f;
    s32 depth_clip_enable = 1;
    s32 scissor_enable = 0;
    s32 multisample = 0;
    s32 aa_lines = 0;
    rasteriser_state_creation_params(){};
};
```

If we supply `PEN_FILL_WIREFRAME` as the `fill_mode` then in the OpenGL implementation we can intercept this value and handle it differently. If we target OpenGLES then the internal raster state sets a flag which says we want to draw with wireframe, at this point the fill mode is still `GL_FILL` and not `GL_LINE` like it would be on regular OpenGL.

```c++
void direct::renderer_create_rasterizer_state(const rasteriser_state_creation_params& rscp, u32 resource_slot)
{
    // ...
    
    rs.polygon_mode = to_gl_polygon_mode(rscp.fill_mode);
    rs.gles_wireframe = false;
#ifdef PEN_GLES3
    if(rscp.fill_mode == PEN_FILL_WIREFRAME)
        rs.gles_wireframe = true;
#endif
}
```

When a draw call is made the raster state is set from a higher level and stored in an internal state caching mechanism, so at the time we want to call `glDrawElements` and friends we can check if we expect wireframe mode and switch to using `GL_LINES` as the primitive topology of the draw call:

```c++
u32 _gles_wireframe(u32 primitve_topology)
{
    if(primitve_topology != GL_LINES && primitve_topology != GL_LINE_STRIP)
        if(_res_pool[s_state.raster_state].raster_state.gles_wireframe)
            return GL_LINE_STRIP;
    return primitve_topology;
}
#ifdef PEN_GLES3
#define PEN_GLES_WIREFRAME_TOPOLOGY(pt) _gles_wireframe(pt)
#else
#define PEN_GLES_WIREFRAME_TOPOLOGY(pt) pt
#endif

void direct::renderer_draw(u32 vertex_count, u32 start_vertex, u32 primitive_topology)
{
    primitive_topology = to_gl_primitive_topology(primitive_topology);
    bind_state(primitive_topology);
    primitive_topology = PEN_GLES_WIREFRAME_TOPOLOGY(primitive_topology);

    CHECK_CALL(glDrawArrays(primitive_topology, start_vertex, vertex_count));
	
	// ..
}
```

I wrapped it in a macro to completely omit the code on normal OpenGL platforms so as to not incur any additional performance overhead. With this new code enabled here is the outcome:

![working](/images/posts/wireframe/constraints-wireframe-working.gif)

The solution is by no means perfect - it will not emulate the exact behaviour of proper `GL_LINE` fill mode. I had to use a line strip because otherwise some edges are not generated by using a line list, and this means that depending on the triangle and index order you may get lines joining vertices in a mesh that do not match the silhouette, but for convex meshes you should get pretty good results. Concave meshes may suffer the artefact of joining lines cutting through some of the concavity of the meshes. Having said that, for my cases this is good enough and it makes the sample look much nicer and hits parity with the rest of the platforms. Here is a more complex scene with some meshes which have denser vertex distribution:

![rb](/images/posts/wireframe/rb-wireframe.gif)

Despite it's deprecation on Apple platforms and my preference toward other (more modern) rendering API's, OpenGL lives on and I am still supporting it, this was a nice fix to have in the bag and maybe some other people find it useful as well. Let me know if you do!

