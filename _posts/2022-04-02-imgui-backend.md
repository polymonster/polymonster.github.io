---
title: 'Dear ImGui backend implementation with docking and viewports in Rust'
date: 2022-04-01 14:00:00
---

Following on from my previous post about building a new graphics engine in Rust, I have made decent progress in implementing a feature-complete [Dear ImGui](https://github.com/ocornut/imgui) rendering and platform backend using my engine [hotline](https://github.com/polymonster/hotline) `gfx::` and `os::` APIs. It has viewports (multiple windows), docking, mouse cursors and input. I haven’t yet looked into improving the front end and making the API more ergonomic to use in Rust, but I thought now would be a good time to cover some of the challenges I faced while they are fresh in my mind.

This was a fun task to complete, filling out backend implementations is like colouring in - you don’t have to make too many decisions because the API is already structured. It was easy to jump in and do a half an hour of coding here and there until the task was completed. Here is a gif of it in action:

![github](/images/posts/rust-gfx/viewports_complete.gif)

## A love letter to Dear ImGui

I have been using [Dear ImGui](https://github.com/ocornut/imgui) for quite some time. I have used it in every place I have worked since about 2014 and from that point it revolutionised programming for me completely. The ease of use and ability to quickly hook up UI to parameters in code made development so much more rapid and expressive. Before this we toiled away with C# wrappers for UI code and all this plumbing required piping values from C++ to tweak them in a WinForm and back again. Now ImGui dominates the landscape, pretty much every hobby engine project you see on GitHub has it integrated and even the big dogs such as Intel, Nvidia and many more have open source tools available which use ImGui. I think ImGui is an excellent example of C++ code; bloat free, minimal dependencies, simple and direct… yet it has the power to create truly [magical](https://github.com/wolfpld/tracy) and [complex](https://www.nvidia.com/en-gb/omniverse/apps/view/) [things](https://github.com/HankiDesign/awesome-dear-imgui). It generates data that makes draw calls extremely efficient and decouples that from the gui calls themselves. For anyone reading this who is not familiar with the C++ coding style of ImGui, I could not recommend it enough, the backend implementations are great too. So thanks [Omar](https://twitter.com/ocornut) for all the work put into Dear ImGui. It truly is a masterpiece.

I should point out that Dear ImGui is not the only available option, to name a few there is [nuklear](https://github.com/vurtun/nuklear) and [microgui](https://github.com/rxi/microui) and for Rust maybe more fittingly [egui](https://github.com/emilk/egui). I am sticking with Dear ImGui for now though, just because of how much I have used it and my familiarity with it. I have implemented a few backend implementations before in C++ so that makes it a bit easier to focus on the details of how this integrates better with Rust.

## Docking and Viewports with imgui-sys

There is already the [imgui-rs](https://github.com/imgui-rs/imgui-rs) repository on GitHub, which is the most popular and active ImGui implementation. This does not support docking or viewports yet because they are not in the main upstream repository of ImGui, instead being inside the docking branch. These are crucial features that I want. Docking improves ImGui a lot in my opinion; before docking we had to use columns or child regions for certain types of layouts that can now be replaced by docking, which allows windows to be docked inside one another to create complex configurations. Viewports adds the ability to drag windows around on your desktop and have more than one native operating system window, which is great for multiple monitor environments and further allows you to create more complicated window setups that are user driven. 

I am using the [imgui-sys](https://github.com/imgui-rs/imgui-rs/tree/main/imgui-sys) bindings from within the imgui-rs repository, these are Rust FFI bindings to [cimgui](https://github.com/cimgui/cimgui) which is a C wrapper API around C++ Dear ImGui. I had trouble using imgui-sys directly from crates.io; I wasn’t able to enable the docking feature, so currently I have opted to include the imgui-sys source directly inside my repository. A really cool feature of having the source code inside the repository I discovered is that it is possible to modify the C++ code and it gets automatically rebuilt with `cargo build`; this really blew my mind and opens up great opportunities for C++ and Rust interop.

## Getting Started

Previously I had been dealing with unsafe APIs through [windows-rs](https://github.com/microsoft/windows-rs) and had become quite comfortable in doing so. I had become confident in using unsafe code and providing a safe wrapper around it for users. My newfound skills did not prepare me for using the raw [imgui-sys](https://github.com/imgui-rs/imgui-rs/tree/main/imgui-sys) bindings however, and this was a more complex challenge altogether. I now appreciate how nice the windows APIs for Rust are, functions return `Result`s which can be bubbled up, the typing is fairly strong and the APIs generally do not require use of static mutable state. Creating Win32 COM objects and lifetime ownership ties directly to Rust’s `Drop` so the API fits quite nicely. ImGui on the other hand is very C-style in its implementation, which is at odds with best practices for Rust because ImGui uses a lot of static mutable state and heap allocated memory. My job here is to abstract these issues to provide a safe API for other users.

## Basic Structure

ImGui is structured to allow different platform specific implementations to be used, it has two backends; renderer ie. (Direct3D or OpenGL) and platform ie. (Win32 or NS). Here I am opting to use my own abstraction APIs `gfx::` for renderer and `os::` for platform. This way if I create multiple implementations of these APIs I should automatically get ImGui support. So for example currently I only have one implementation of `gfx::`, which is for Direct3D12, if I decided to add a new implementation using Metal for macOS then my ImGui implementation will work on either Direct3D12 or Metal through the power of abstraction.

The renderer implementation is responsible for rendering the gui windows and widgets and the platform implementation is responsible for creating operating system windows, managing input and mouse cursors. ImGui comes with *tons* of example backend [implementations](https://github.com/ocornut/imgui/tree/master/backends), these are really great and just generally good examples of platform abstractions whether you used them for ImGui or not. I used the Win32 and Direct3D12 ImGui C++ examples for reference and allowed these to shape primarily my `os::` API. I had previously used the Direct3D12 example code from ImGui to start the `gfx::` API, which at this point is mostly complete. The ImGui examples are just a nice reference for bare bones minimal implementations of getting something rendering on screen.

To hook in an ImGui implementation there are not many functions that need to be implemented in the public facing API:
- Setup
- New Frame
- Render

In my application I already have created a window, a device, and a swap chain that can be rendered to. These are passed to ImGui at start-up and the `new_frame` and `render` functions can be hooked into the main application loop.

```rust
// pass a device, swap chain and main window when creating an imgui instance
let mut imgui_info = imgui::ImGuiInfo {
    device: &mut dev,
    swap_chain: &mut swap_chain,
    main_window: &win,
    fonts: vec![asset_path
        .join("..\\..\\samples\\imgui_demo\\Roboto-Medium.ttf")
        .to_str()
        .unwrap()
        .to_string()],
};
let mut imgui = imgui::ImGui::create(&mut imgui_info).unwrap();

// main loop
while app.run() {
    // ..

    // begin pass on command buffer
    let mut pass = swap_chain.get_backbuffer_pass_mut();
    cmdbuffer.begin_render_pass(&mut pass);

    // new frame, pass app, window and device 
    imgui.new_frame(&mut app, &mut win, &mut dev);
    
    // imgui calls go here

    // render pass app, window, device and cmdbuffer 
    imgui.render(&mut app, &mut win, &mut dev, &mut cmdbuffer);
}
```

## Renderer Backend

I first began work on the renderer backend; until the viewports feature is required a platform backend is not required, and with my `gfx::` API wrapper around Direct3D12 there is only minimal code required.

### Setup

In my case this all happens in the `ImGui::create()` function, I decided here to keep all of my related ImGui state and objects inside an `ImGui` instance and steer away from the C-Style static state that the C++ ImGui implementation goes for. In the info struct we pass the main window, the device and swap chain for the main window. These objects are necessary to be able to create new `gfx::` and `os::` objects for use during rendering.

We need to create a few basic things in order to draw the data generated by ImGui: a pipeline with all appropriate render state and a simple vertex and fragment shader, a vertex and index buffer for each image in the swap chain that we rotate through each frame so that we do not overwrite a buffer that’s in flight on the GPU, and finally a texture containing font glyphs. The fonts are generated from a TrueType font file and in my case I am using the `stb_font` implementation that generates the glyphs into texture data.

```rust
// imgui instance holds objects required for rendering
pub struct ImGui<D: Device, A: App> {
    _native_handle: A::NativeHandle,
    font_texture: D::Texture,
    pipeline: D::RenderPipeline,
    // a set of buffers for each buffer in the swap chain
    buffers: Vec<RenderBuffers<D>>,
    last_cursor: os::Cursor,
}

// buffers have size to track and resize them
#[derive(Clone)]
struct RenderBuffers<D: Device> {
    vb: D::Buffer,
    ib: D::Buffer,
    vb_size: i32,
    ib_size: i32,
}

impl<D, A> ImGui<D, A> where D: Device, A: App {
    pub fn create(info: &mut ImGuiInfo<D, A>) -> Result<Self, gfx::Error> {
        unsafe {
            igCreateContext(std::ptr::null_mut());
            let mut io = &mut *igGetIO();

            // add fonts
            for font_name in &info.fonts {
                let null_font_name = CString::new(font_name.clone()).unwrap();
                ImFontAtlas_AddFontFromFileTTF(
                    io.Fonts,
                    null_font_name.as_ptr() as *const i8,
                    16.0,
                    std::ptr::null_mut(),
                    std::ptr::null_mut(),
                );
            }

            // ..
            // setup input

            // texture
            let font_tex = create_fonts_texture::<D>(&mut info.device)?;

            // pipeline 
            let pipeline = create_render_pipeline(info)?;

            // create render buffers
            let mut buffers: Vec<RenderBuffers<D>> = Vec::new();
            let num_buffers = (*info.swap_chain).get_num_buffers();
            for _i in 0..num_buffers {
                buffers.push(RenderBuffers {
                    vb: create_vertex_buffer::<D>(&mut info.device, DEFAULT_VB_SIZE)?,
                    vb_size: DEFAULT_VB_SIZE,
                    ib: create_index_buffer::<D>(&mut info.device, DEFAULT_IB_SIZE)?,
                    ib_size: DEFAULT_IB_SIZE,
                })
            }

            // ..
            // enum monitors

            // ..
            // setup callbacks

            Ok(imgui)
        }
    }
```

### New Frame

We call the `new_frame` function every frame, it doesn’t do anything related to rendering it’s more related to the platform implementation but after calling it we can make ImGui calls to build windows and UI’s, it’s at this point where ImGui takes over and does all the hard work for us generating command lists which can be drawn efficiently. The command lists primarily consist of vertex and index data that we copy into buffers for rendering.

### Render

This is where the magic happens. The main function to implement is called `render_draw_data`. We get passed the `ImDrawData` structure, which contains command lists and buffers that we can make draw calls from. 

```rust
fn render_draw_data<D: Device>(
    draw_data: &ImDrawData,
    device: &mut D,
    cmd: &mut D::CmdBuf,
    buffers: &mut Vec<RenderBuffers<D>>,
    pipeline: &D::RenderPipeline,
    font_texture: &D::Texture,
) -> Result<(), gfx::Error>
```

Per-frame we have just a single vertex buffer and index buffer that we use during rendering. The vertex and index buffers may need to be resized to accommodate the contents of the draw data, so the buffer sizes are tracked and new buffers created if the old ones are not large enough.

```rust
let mut buffers = &mut buffers[bb];

// resize vb
if draw_data.TotalVtxCount > buffers.vb_size {
    buffers.vb = create_vertex_buffer::<D>(device, draw_data.TotalVtxCount)?;
    buffers.vb_size = draw_data.TotalVtxCount;
}

// resize ib
if draw_data.TotalIdxCount > buffers.ib_size {
    buffers.ib = create_index_buffer::<D>(device, draw_data.TotalIdxCount)?;
    buffers.ib_size = draw_data.TotalIdxCount;
}
```

Once we have enough space in the vertex and index buffers they are updated with the latest data for the frame. We iterate over the command lists and copy the vertex and index data for each command list into the gpu buffers.

```rust
// update buffers
let imgui_cmd_lists =
    std::slice::from_raw_parts(draw_data.CmdLists, draw_data.CmdListsCount as usize);
let mut vertex_write_offset = 0;
let mut index_write_offset = 0;

for imgui_cmd_list in imgui_cmd_lists {
    // vertex
    let draw_vert = &(*(*imgui_cmd_list)).VtxBuffer;
    let vb_size_bytes = draw_vert.Size as usize * std::mem::size_of::<ImDrawVert>();
    let vb_slice = std::slice::from_raw_parts(draw_vert.Data, draw_vert.Size as usize);
    buffers.vb.update(vertex_write_offset, vb_slice)?;
    vertex_write_offset += vb_size_bytes as isize;
    // index
    let draw_index = &(*(*imgui_cmd_list)).IdxBuffer;
    let ib_size_bytes = draw_index.Size as usize * std::mem::size_of::<ImDrawIdx>();
    let ib_slice = std::slice::from_raw_parts(draw_index.Data, draw_index.Size as usize);
    buffers.ib.update(index_write_offset, ib_slice)?;
    index_write_offset += ib_size_bytes as isize;
}
```

Next we set up some of the basic render state for the pass, binding the vertex and index buffer, setting a viewport and pushing some constants for the shader. The push constants are simply an orthographic matrix.

```rust
// update push constants
let l = draw_data.DisplayPos.x;
let r = draw_data.DisplayPos.x + draw_data.DisplaySize.x;
let t = draw_data.DisplayPos.y;
let b = draw_data.DisplayPos.y + draw_data.DisplaySize.y;

let mvp: [[f32; 4]; 4] = [
    [2.0 / (r - l), 0.0, 0.0, 0.0],
    [0.0, 2.0 / (t - b), 0.0, 0.0],
    [0.0, 0.0, 0.5, 0.0],
    [(r + l) / (l - r), (t + b) / (b - t), 0.0, 1.0],
];

// marker for PIX
cmd.set_marker(0xff00ffff, "ImGui");

let viewport = gfx::Viewport {
    x: 0.0,
    y: 0.0,
    width: draw_data.DisplaySize.x,
    height: draw_data.DisplaySize.y,
    min_depth: 0.0,
    max_depth: 1.0,
};

// set state
cmd.set_viewport(&viewport);
cmd.set_vertex_buffer(&buffers.vb, 0);
cmd.set_index_buffer(&buffers.ib);
cmd.set_render_pipeline(&pipeline);
cmd.push_constants(0, 16, 0, &mvp);
```

After the basic render state is set up we can begin making draw calls. Iterating over command lists and then command buffers, we make a draw call per command buffer where we may swap texture and supply different scissor rectangles for clipping geometry.

```rust
let clip_off = draw_data.DisplayPos;
let mut global_vtx_offset = 0;
let mut global_idx_offset = 0;
for imgui_cmd_list in imgui_cmd_lists {
    let imgui_cmd_buffer = (**imgui_cmd_list).CmdBuffer;
    let imgui_cmd_data = std::slice::from_raw_parts(imgui_cmd_buffer.Data, imgui_cmd_buffer.Size as usize);
    let draw_vert = &(*(*imgui_cmd_list)).VtxBuffer;
    let draw_index = &(*(*imgui_cmd_list)).IdxBuffer;
    for i in 0..imgui_cmd_buffer.Size as usize {
        let imgui_cmd = &imgui_cmd_data[i];
        if imgui_cmd.UserCallback.is_some() {
            // ..
        } 
        else {
            // ..

            cmd.set_render_heap(1, device.get_shader_heap(), font_tex_index);
            cmd.set_scissor_rect(&scissor);
            cmd.draw_indexed_instanced(
                imgui_cmd.ElemCount,
                1,
                imgui_cmd.IdxOffset + global_idx_offset,
                (imgui_cmd.VtxOffset + global_vtx_offset) as i32,
                0,
            );
        }
    }
    global_idx_offset += draw_index.Size as u32;
    global_vtx_offset += draw_vert.Size as u32;
}
```

It really is that simple! Just a few basic things that when fed the data from ImGui can create wonderful and complex things. I can’t under-state how happy this makes me; bare minimal code that has powerful and almost magical results. No bloat, just a few hundred lines of code and you have something which will give you so, so much.

## Platform Backend

With the renderer backend complete, enabling docking works and this allows us to render ImGui data to a single physical operating system window, which we pass the swap chain of and render into in the render function. To enable viewports, things get a bit more complicated and stray into more tricky territory. Viewports mean we can drag the ImGui windows between different monitors or position them anywhere on the desktop; they are no longer constrained to the primary window we created at startup.

Before implementing viewports there are a few bits and pieces necessary for a non-viewport ImGui implementation. This mostly consists of forwarding window events (key-presses, mouse state, etc). I had to implement most of the events in the `wndproc` for my `os::` API. I used the ImGui Win32 backend here for reference. One thing I intend to revisit is that I had introduced static mutable state inside the `win32::` implementation because of the wndproc’s callback nature. I think I can intercept the events earlier and keep the state inside my `App` struct and avoid the synchronisation of static mutable state.

During setup I added functionality to enumerate the display monitors and obtain their DPI, and also needed to set up key mappings. When `new_frame` is called here, we simply pass the state of the keyboard and mouse to ImGui so it can use the inputs.

### Callbacks

ImGui provides a series of callback functions which need hooking up to enable the viewport implementation. And then it is just a case of implementing all of the callbacks to achieve the full ImGui implementation. 

```rust
let platform_io = &mut *igGetPlatformIO();

// platform hooks
platform_io.Platform_CreateWindow = Some(platform_create_window::<D, A>);
platform_io.Platform_DestroyWindow = Some(platform_destroy_window::<D, A>);
platform_io.Platform_ShowWindow = Some(platform_show_window::<D, A>);
platform_io.Platform_SetWindowPos = Some(platform_set_window_pos::<D, A>);
platform_io.Platform_SetWindowSize = Some(platform_set_window_size::<D, A>);
platform_io.Platform_SetWindowFocus = Some(platform_set_window_focus::<D, A>);
platform_io.Platform_GetWindowFocus = Some(platform_get_window_focus::<D, A>);
platform_io.Platform_GetWindowMinimized = Some(platform_get_window_minimised::<D, A>);
platform_io.Platform_SetWindowTitle = Some(platform_set_window_title::<D, A>);
platform_io.Platform_GetWindowDpiScale = Some(platform_get_window_dpi_scale::<D, A>);
platform_io.Platform_UpdateWindow = Some(platform_update_window::<D, A>);

// render hooks
platform_io.Renderer_RenderWindow = Some(renderer_render_window::<D, A>);
platform_io.Renderer_SwapBuffers = Some(renderer_swap_buffers::<D, A>);

// need to hook these c-compatible getter funtions due to complex return types
ImGuiPlatformIO_Set_Platform_GetWindowPos(platform_io, platform_get_window_pos::<D, A>);
ImGuiPlatformIO_Set_Platform_GetWindowSize(platform_io, platform_get_window_size::<D, A>)
```

This is easier said than done. Callbacks work nice and easily, and to my delight even generics can be supplied (I will get to the reason for this later). The main difficulties I encountered here were how to deal with the `ImViewportData` and pass around objects with lifetimes.

ImGui requires `ImViewportData` to be a non-null pointer for any windows it expects to be viewport windows, otherwise it will assert.  I opted to go for heap allocated memory and to essentially leave the ownership of this data to ImGui. Initially I tried a few things and ended up with difficulties because I was taking pointers within structs and sometimes this may have got moved. In order to allocate memory I use `std::alloc` and zero the memory. I made some utility functions to create new objects. There are two structures which are required:

```rust
struct ViewportData<D: Device, A: App> {
    /// if viewport is main, we get the window from UserData and the rest of this struct is null
    main_viewport: bool,
    window: Vec<A::Window>,
    swap_chain: Vec<D::SwapChain>,
    cmd: Vec<D::CmdBuf>,
    buffers: Vec<RenderBuffers<D>>,
}

struct UserData<'a, D: Device, A: App> {
    app: &'a mut A,
    device: &'a mut D,
    main_window: &'a mut A::Window,
    pipeline: &'a D::RenderPipeline,
    font_texture: &'a D::Texture,
}

// function to create heap allocated ViewportData
fn new_viewport_data<D: Device, A: App>() -> *mut ViewportData<D, A> {
    unsafe {
        let layout =
            std::alloc::Layout::from_size_align(std::mem::size_of::<ViewportData<D, A>>(), 8).unwrap();
        std::alloc::alloc_zeroed(layout) as *mut ViewportData<D, A>
    }
}
```

For the main window we can create an initial `ViewportData` when we call `imgui::create()` and here the ownership and borrowing is quite straightforward. In order to create a `SwapChain` or `CmdBuffer` we need a `Device` and to create a `Window` we need an `App`. We borrow a `Device` and `App` when calling the `imgui::create()` function so this is simple. It becomes more difficult when we need to create a `ViewportData` from inside one of the callback functions.

To solve this issue I introduced the `UserData` struct and passed borrowed references to the `Device` and `App` and then assigned the ImGui `io.UserData` pointer to my `UserData` struct. The platform callback functions always get called from a call stack either beneath the `new_frame` or `render` functions. This gives me the ability to borrow `Device` and `App` into these functions, pack them into the struct, assign the pointer and then dereference the pointer from the callback functions.

```rust
pub fn new_frame(
    &mut self,
    app: &mut A,
    main_window: &mut A::Window,
    device: &mut D,
) {
    let size = main_window.get_size();

    unsafe {
        let io = &mut *igGetIO();

        // gotta pack the refs into a pointer and into UserData for callbacks
        let mut ud = UserData {
            device: device,
            app: app,
            main_window: main_window,
            pipeline: &self.pipeline,
            font_texture: &self.font_texture,
        };
        io.UserData = (&mut ud as *mut UserData<D, A>) as _;

        // updates happen here, callbacks may be called. UserData is on the stack, we have borrowed refs to app, main_window, device
        igNewFrame();

        // return io.UserData to null as we will not have the references after this function completes
        io.UserData = std::ptr::null_mut();
    }
}
```

The implementation of the callbacks was fairly straight forward and they map quite directly to functions in the Win32 API. I added the corresponding functions to my `os::Window` trait and supplied some `From` traits to help with the conversions. I also added utility functions to help unpacking the `UserData` and `ViewportData` from the pointers supplied by ImGui.

```rust
/// handles the case where we can return an imgui created window from ViewportData, or borrow the main window
/// from UserData
fn get_viewport_window<'a, D: Device, A: App>(vp: *mut ImGuiViewport) -> &'a mut A::Window {
    unsafe {
        let vp_ref = &mut *vp;
        let vd = &mut *(vp_ref.PlatformUserData as *mut ViewportData<D, A>);
        if vd.main_viewport {
            let io = &mut *igGetIO();
            let ud = &mut *(io.UserData as *mut UserData<D, A>);
            return ud.main_window;
        }
        return &mut vd.window[0];
    }
}

// one of the simpler examples.
unsafe extern "C" fn platform_get_window_pos<D: Device, A: App>(vp: *mut ImGuiViewport, out_pos: *mut ImVec2) {
    let window = get_viewport_window::<D, A>(vp);
    let pos = window.get_pos();
    (*out_pos).x = pos.x as f32;
    (*out_pos).y = pos.y as f32;
}
```

## Generics

The code snippets posted contain a fully generic compile-time implementation. So it would be theoretically possible to have multiple ImGui implementations running side by side with different graphics `Devices`. When I first implemented this I did not use generics and instead just had a compile time `use`.

```rust
 #[cfg(target_os = "windows")]
 use crate::os::win32 as os_platform;
 use crate::gfx::d3d12 as gfx_platform;
```

Then I would pass to functions or store in structs and have an implementation like so:

```rust
// struct 
pub struct ImGuiInfo<'a> {
     pub device: &'a mut gfx_platform::Device,
     pub swap_chain: &'a mut gfx_platform::SwapChain,
     pub main_window: &'a os_platform::Window,
     pub fonts: Vec<String>,
}

// function
fn create_fonts_texture(
     device: &mut gfx_platform::Device,
 ) -> Result<gfx_platform::Texture, gfx::Error>

 // ImGui impl
 impl ImGui {
     pub fn create(info: &mut ImGuiInfo) -> Result<Self, gfx::Error> {
```

The problem with this approach is that only a single `os::` and `gfx::` backend could be used for ImGui. It might be unlikely that I would run multiple ImGui implementations side by side, but the possibility of doing so is a nice option to have. Mostly I didn’t like this approach because elsewhere in my code I was already doing the similar compile time directive to select which `os::` and which `gfx::` backend to use.

I wanted this decision to happen in the main entry point of the application, and for that combination of backend choices to propagate through the ImGui implementation. Switching from the single compile time backend selection to a generic version didn’t take too long, but it was a bit of work to untangle and once you start pulling the thread, everything is erroring and nothing will compile and it spread into the `SwapChain` and the `App`. But once complete I now have stripped it down to a single place where the backends are selected.

The commit for this change is [here](https://github.com/polymonster/hotline/commit/2198a73ca5ea37088b7e5090ec62184bc21f6271) so you can take a look at what it took for yourself. But the main gist of it is:

```rust
// struct
pub struct ImGuiInfo<'a, D: Device, A: App> {
     pub device: &'a mut D,
     pub swap_chain: &'a mut D::SwapChain,
     pub main_window: &'a A::Window,
     pub fonts: Vec<String>,
}

// function
fn create_fonts_texture<D: Device>(
     device: &mut D,
 ) -> Result<D::Texture, gfx::Error>

 // and the ImGui impl using 'where'
impl<D, A> ImGui<D, A> where D: Device, A: App {
     pub fn create(info: &mut ImGuiInfo<D, A>) -> Result<Self, gfx::Error> {
```

And as mentioned earlier, I was happy to find that generics can be supplied to the callback functions required by ImGui:

```rust
// assign callback with generics
platform_io.Platform_CreateWindow = Some(platform_create_window::<D, A>);

// callback function which requires generics
unsafe extern "C" fn platform_create_window<D: Device, A: App>(vp: *mut ImGuiViewport) {
    // ..
}
```

The only issue I had here was when I made the `Device::create_swap_chain()` generic to accept the `App` instead of a fixed `win32::` platform for `d3d12::`. In the function implementation it requires obtaining the `HWND` handle from the internal `win32::Window` but since it was now an `os::Window` trait I only have access to the trait members and do not want to expose `HWND` to the public API. I have only briefly looked into this and some some information relating to `downcast_ref`, but this was in the context of a `dyn` object. In order to circumvent this for the time being I added a function which can get the `HWND` as an `isize` (which is actually what it is under the hood anyway). So I am able to work around this for now, but I will need to do some more research in how to do this for more complex types should I need it in the future. 

## Next Steps - Frontend?

I implemented the full ImGui demo application. The code is somewhat hideous and not very ergonomic, so I will be planning to find solutions to provide a nicer frontend to make ImGui more usable. I mentioned [imgui-rs](https://github.com/imgui-rs/imgui-rs/tree/main/imgui-sys) earlier, so that is an option, however it is still not officially supporting docking so I’m not sure what it would take to integrate what I have with what is already there.
