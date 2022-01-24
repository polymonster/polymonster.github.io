---
title: WebGL Demos
subtitle: powered by pmtech.
description: powered by pmtech.
featured_image: /images/site/spacefilling.png
---

These demos are from my game engine [pmtech](https://github.com/polymonster/pmtech) they are built using WebAssembly and WebGL with the help of emscripten. You can find out more information about the example projects on the pmtech [wiki](https://github.com/polymonster/pmtech/wiki/Examples).

For web browsers Chrome, Firefox and Microsoft Edge are all supported. Support on Safari is available if you enable the experimental feature "WebGL 2.0" but without "WebGL via Metal". Mobile browser support may be more limited.

These samples are currently work in progress with some missing features compared to the native pmtech implementations, the reference images were generated on macOS/Metal.

Controls:
- [CTRL or CMD + X] - Toggle imgui "dev_ui".
- [Alt or Option + Left Mouse Drag] - Rotate camera.
- [CTRL or CMD + Left Mouse Drag] - Pans camera (macOS).
- [CTRL or CMD + Middle Mouse Drag] - Pans camera (Windows / Linux).
- [Mouse Wheel] -  Zoom camera.
- [F] - Focus camera on selected object.
- [Left Click] - Select items, or grab physics objects.

<style>
.row {
  display: flex;
  flex-wrap: wrap;
  padding: 0 4px;
}

/* Create four equal columns that sits next to each other */
.column {
  flex: 25%;
  max-width: 25%;
  padding: 0 4px;
}

.column img {
  margin-top: 8px;
  vertical-align: middle;
  width: 100%;
}

.pad {
  padding-top: 50px;
}
</style>
<p></p>
<p></p>

<div class="row">
  <div class="column">
    <a href="http://www.polymonster.co.uk/pmtech/examples/clear.html"><img src="/images/pmtech/thumbs/clear.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/render_target.html"><img src="/images/pmtech/thumbs/render_target.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/depth_texture.html"><img src="/images/pmtech/thumbs/depth_texture.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/texture_formats.html"><img src="/images/pmtech/thumbs/texture_formats.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/geometry_primitives.html"><img src="/images/pmtech/thumbs/geometry_primitives.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/audio_player.html"><img src="/images/pmtech/thumbs/audio_player.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/multiple_render_targets.html"><img src="/images/pmtech/thumbs/multiple_render_targets.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/physics_constraints.html"><img src="/images/pmtech/thumbs/physics_constraints.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/skinning.html"><img src="/images/pmtech/thumbs/skinning.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/post_processing.html"><img src="/images/pmtech/thumbs/post_processing.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/entities.html"><img src="/images/pmtech/thumbs/entities.jpg"></a>
  </div>
  <div class="column">
    <a href="http://www.polymonster.co.uk/pmtech/examples/basic_triangle.html"><img src="/images/pmtech/thumbs/basic_triangle.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/buffer_multi_update.html"><img src="/images/pmtech/thumbs/buffer_multi_update.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/debug_text.html"><img src="/images/pmtech/thumbs/debug_text.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/blend_modes.html"><img src="/images/pmtech/thumbs/blend_modes.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/cubemap.html"><img src="/images/pmtech/thumbs/cubemap.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/shader_toy.html"><img src="/images/pmtech/thumbs/shader_toy.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/maths_functions.html"><img src="/images/pmtech/thumbs/maths_functions.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/complex_rigid_bodies.html"><img src="/images/pmtech/thumbs/complex_rigid_bodies.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/vertex_stream_out.html"><img src="/images/pmtech/thumbs/vertex_stream_out.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/sss.html"><img src="/images/pmtech/thumbs/sss.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/area_lights.html"><img src="/images/pmtech/thumbs/area_lights.jpg"></a>
  </div>
  <div class="column">
    <a href="http://www.polymonster.co.uk/pmtech/examples/basic_texture.html"><img src="/images/pmtech/thumbs/basic_texture.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/texture_array.html"><img src="/images/pmtech/thumbs/texture_array.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/input_example.html"><img src="/images/pmtech/thumbs/input_example.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/stencil_buffer.html"><img src="/images/pmtech/thumbs/stencil_buffer.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/volume_texture.html"><img src="/images/pmtech/thumbs/volume_texture.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/render_target_mip_maps.html"><img src="/images/pmtech/thumbs/render_target_mip_maps.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/single_shadow.html"><img src="/images/pmtech/thumbs/single_shadow.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/instancing.html"><img src="/images/pmtech/thumbs/instancing.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/shadow_maps.html"><img src="/images/pmtech/thumbs/shadow_maps.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/pmfx_renderer.html"><img src="/images/pmtech/thumbs/pmfx_renderer.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/stencil_shadows.html"><img src="/images/pmtech/thumbs/stencil_shadows.jpg"></a>
  </div>
  <div class="column">
    <a href="http://www.polymonster.co.uk/pmtech/examples/basic_compute.html"><img src="/images/pmtech/thumbs/basic_compute.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/depth_test.html"><img src="/images/pmtech/thumbs/depth_test.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/imgui_example.html"><img src="/images/pmtech/thumbs/imgui_example.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/rasterizer_state.html"><img src="/images/pmtech/thumbs/rasterizer_state.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/play_sound.html"><img src="/images/pmtech/thumbs/play_sound.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/msaa_resolve.html"><img src="/images/pmtech/thumbs/msaa_resolve.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/rigid_body_primitives.html"><img src="/images/pmtech/thumbs/rigid_body_primitives.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/cull_sort.html"><img src="/images/pmtech/thumbs/cull_sort.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/sdf_shadow.html"><img src="/images/pmtech/thumbs/sdf_shadow.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/dynamic_cubemap.html"><img src="/images/pmtech/thumbs/dynamic_cubemap.jpg"></a>
    <a href="http://www.polymonster.co.uk/pmtech/examples/global_illumination.html"><img src="/images/pmtech/thumbs/global_illumination.jpg"></a>
  </div>
</div>