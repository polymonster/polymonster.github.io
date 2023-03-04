---
title: 'Building a new graphics engine in Rust - Part 3'
date: 2023-03-03 00:00:00
---

Following on from the [part 2](https://www.polymonster.co.uk/blog/bulding-new-engine-in-rust-2) post a little while ago, I have been continuing work on my graphics engine [hotline](https://github.com/polymonster/hotline) in Rust. My recent focus has been on plugins, multi-threaded command buffer generation and hot reloading for Rust code, `hlsl` shader code and `pmfx` render configs. I have made decent progress to the point where there is something quite usable and structured in a way I am relatively happy with. This leg of the journey has been by far the most challenging though, so I wanted to write about my current progress and detail some of the issues I have faced.

Here are the results of my first sessions actually using the engine. I created these primitives and used the hot reloading system to iterate on them to get perfect vertices, normals, and uv-coordinates. My intention is to use this tool for graphics demos and procedural generation so the focus is on making a live coding environment and not an interactive GUI editor. The visual `client` provides some feedback to the user and information but it is not really an editor as such, the editing goes in source code and data files which is reflected in the `client`. In time I may decide to add more interactive editor features but for now it's all about coding. Here's a demo video of some of the features:

<iframe width="560" height="315" src="https://www.youtube.com/embed/jkD78gXfIe0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

I am using a single screen with vscode on the left and hotline on the right; by launching the hotline client executable from the vscode terminal it prints errors for hot reloading in the terminal and the line number links are clickable to automatically go to error lines.

## Recap

I had previously created `gfx`, `os` and `av` abstraction API’s that currently have Windows-specific backend implementations, but the API’s are designed to easily add more platforms in the future. At this time I am trying to push as far ahead as possible on a single platform because I did spend a lot of time working on cross platform support in my C++ [game engine](https://github.com/polymonster/pmtech) or the engines I have worked on for my day job. Cross platform maintenance can become time consuming, so for a little while I have decided just to focus on feature development.

I have also been on a few side quests that have fallen under the umbrella of this graphics engine project, but I wrote about those separately. They were implementing [imgui](https://www.polymonster.co.uk/blog/imgui-backend) with viewports and docking, and [maths-rs](https://github.com/polymonster/maths-rs) a linear algebra library I have been working on while away from my Windows desktop machine. A few people asked about maths-rs and why don’t I just use any existing library? I simply wanted something to work on using my laptop in the spare time I had and a maths library was the first thing I thought of. This is a downside of having a Windows only engine, my opportunity to work on it is limited to being chained to a machine in my house. I already had a C++ [maths library](https://github.com/polymonster/maths) that I ported a lot of the code from, but while I was there I improved the API consistency, added more overlap, intersection, distance functions to both libraries, and added more tests to assist porting to `maths-rs`. Now I’m at the point where I can use all the libraries to start building graphics demos.

## Crates.io

You can use [hotline](https://github.com/polymonster/hotline) as a library and use the `gfx`, `os`, `av`, `imgui`, `pmfx`, and any other modules it provides. It is now available on [crates.io](https://crates.io/crates/hotline-rs). To my dismay the crates.io registry for the name “hotline” was already taken as a placeholder by someone [else](https://crates.io/search?q=hotline). The same happened to [maths](https://crates.io/search?q=maths), so for both my projects on crates.io I had to call them `hotline-rs` and `maths-rs`. It’s a bit disappointing that people claim the names and then haven’t produced any code yet. I’d be fine with someone claiming the name first if they actually had a decent, usable package.

I had some trouble with crates.io because the package size exceeded the lofty limit of 10mb! Most of my repository size was a result of some executables I am using to build data and shaders. I have these tools from prior work so I am still using the python based build system (built into executables with help from [PyInstaller](https://pyinstaller.org/en/stable/)). To reduce the repository size I moved the executables and data files for the hotline examples into their own GitHub repository [hotline-data](https://github.com/polymonster/hotline-data), which is cloned inside `hotline` as part of a `cargo build`.

This data feature is optional and enabled by default, but it means you could bring your own build system or use the one provided, and more importantly this feature can be disabled from package builds / publishing to crates.io.

I also had some compilation issues when publishing the package to crates.io, because currently Windows is the only supported platform. The core API’s are generic using compile time traits, but samples and plugins need to instantiate a concrete type of GPU `Device` or operating system `Window` and there are no macOS or Linux supported backends yet. In time I would like to add a stub implementation for each module. I worked around this for now by making the entire files that require concrete types as Windows only.

## Plugin-Architecture / Hot Reloading

The main work I have been focusing on for the last few weeks is making live reloadable code work through a plugin system, where plugins can be loaded dynamically at run time with no modifications required to the `client` executable. The `client` provides a very thin wrapper around a main loop, it creates some core resources such as `os::App`, `gfx::Device` and so forth. It provides a core loop that will submit command lists and swap buffers and makes it easy to hook in your own update or render logic. With the `client` running, `plugins` can be dynamically loaded from `dylibs` and code changes can be detected causing the library to be rebuilt and reloaded with the client still running. I am using [hot-lib-reloader](https://crates.io/crates/hot-lib-reloader) to assist the lib reloading, although I need to bypass some of it’s cool features like the `hot_functions_from_file` macro because I wanted to remove the dependency on the `client` knowing about the plugins.

Creating a new plugin is quite easy, first you need a dynamic library crate type. The [plugins](https://github.com/polymonster/hotline/tree/master/plugins) directory in hotline has a few different plugins that can be used as examples. But you basically just need a `Cargo.toml` like this

```toml
[package]
name = "ecs"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["rlib", "dylib"]

[dependencies]
hotline-rs = { path = "../.." }
```

Inside a dynamic library plugin you can choose to get hooked into a few core function calls from the client each frame by implementing the `Plugin` trait:

```rust
use hotline_rs::prelude::*;

pub struct EmptyPlugin;

impl Plugin<gfx_platform::Device, os_platform::App> for EmptyPlugin {
    fn create() -> Self {
        EmptyPlugin {
        }
    }

    fn setup(&mut self, client: Client<gfx_platform::Device, os_platform::App>) 
        -> Client<gfx_platform::Device, os_platform::App> {
        println!("plugin setup");
        client
    }

    fn update(&mut self, client: client::Client<gfx_platform::Device, os_platform::App>)
        -> Client<gfx_platform::Device, os_platform::App> {
        println!("plugin update");
        client
    }

    fn unload(&mut self, client: Client<gfx_platform::Device, os_platform::App>)
        -> Client<gfx_platform::Device, os_platform::App> {
        println!("plugin unload");
        client
    }

    fn ui(&mut self, client: Client<gfx_platform::Device, os_platform::App>)
    -> Client<gfx_platform::Device, os_platform::App> {
        println!("plugin ui");
        client
    }
}

hotline_plugin![EmptyPlugin];
```

The `hotline_plugin!` macro creates a c-abi wrapper around the `Plugin` trait. I initially tried to use a `Box<dyn Plugin>` which was returned from the plugin library to the main client executable so the trait functions could be called, but when trying to lookup the function in the `vtable` memory seemed to be garbage. After some investigation this seems to be because Rust does not have a stable-abi so I created the macro to work around this by allocating the plugin trait on the heap, pass a FFI pointer back to the client and then pass the FFI pointer into a c-abi which the macro generates. I expected with the same compiler that I wouldn't need to make the wrapper API but was unable to get it working.

```rust
#[macro_export]
macro_rules! hotline_plugin {
    ($input:ident) => {
        
        // c-abi wrapper for `Plugin::create`
        #[no_mangle]
        pub fn create() -> *mut core::ffi::c_void {
            let ptr = new_plugin::<$input>() as *mut core::ffi::c_void;
            unsafe {
                let plugin = std::mem::transmute::<*mut core::ffi::c_void, *mut $input>(ptr);
                let plugin = plugin.as_mut().unwrap();
                *plugin = $input::create();
            }
            ptr
        }

        // ..
    }
}
```

## ECS Plugin

All plugins do not necessarily have to implement the `Plugin` trait. Plugins can extend others in custom ways. I started on a basic `ecs` that uses [bevy_ecs](https://docs.rs/bevy_ecs/latest/bevy_ecs/) and the bevy `Scheduler` to distribute work onto different threads. The reason for this `plugin-ception` kind of approach is to be able to edit the core `ecs` while the client is running, as well as extension plugins, but also trying to make the whole thing as flexible as possible to allow freedom to implement totally different types of plugins and be able to work on them with hot-reloading.

To load functions from other libraries you can access the `libs` currently loaded in the hotline `client`. These are just a wrapper around [libloading](https://docs.rs/libloading/latest/libloading/) that allows you to retrieve a `Symbol<T>` by name.

```rust
/// Finds available demo names from inside ecs compatible plugins, call the function `get_system_<lib_name>` to disambiguate
fn get_demo_list(&self, client: &PlatformClient) -> Vec<String> {
    let mut demos = Vec::new();
    for (lib_name, lib) in &client.libs {
        unsafe {
            let function_name = format!("get_demos_{}", lib_name).to_string();
            let list = lib.get_symbol::<unsafe extern fn() ->  Vec<String>>(function_name.as_bytes());
            if let Ok(list_fn) = list {
                let mut lib_demos = list_fn();
                demos.append(&mut lib_demos);
            }
        }
    }
    demos
}
```

The core `ecs` provides some functionality to create `setup`, `update` or `render` systems. You can add your own system functions inside different `plugins` and have the `ecs` plugin locate these systems to build schedules for different `demos`. All system stages get dispatched concurrently on different threads, so in time it’s likely more stages will be added to `setup`, `update` and `render`. Defining custom systems is quite straightforward. These are just `bevy_ecs` systems:

```rust
// update system which takes hotline resources `main_window`, `pmfx` and `app`
#[no_mangle]
fn update_cameras(
    app: Res<AppRes>, 
    main_window: Res<MainWindowRes>,
    mut pmfx: ResMut<PmfxRes>,
    mut query: Query<(&Name, &mut Position, &mut Rotation, &mut ViewProjectionMatrix), With<Camera>>) {    
    let app = &app.0;
    for (name, mut position, mut rotation, mut view_proj) in &mut query {
        // ..
    }

    // ..
}
```

Rendering systems get generated from render graphs specified through the `pmfx` system, and hook themselves into a `bevy_ecs` system function call.

```rust
// render system which takes hotline resource `pmfx` and a `pmfx::View`
#[no_mangle]
pub fn render_meshes(
    pmfx: &bevy_ecs::prelude::Res<PmfxRes>,
    view: &pmfx::View<gfx_platform::Device>,
    mesh_draw_query: bevy_ecs::prelude::Query<(&WorldMatrix, &MeshComponent)>) -> Result<(), hotline_rs::Error> {
    
    // ..
}
```

In order to dynamically locate and call these functions we need to supply a bit of boiler plate to look up the functions by name.

```rust
/// Register demo names for this plugin which is called `ecs_demos`
#[no_mangle]
pub fn get_demos_ecs_demos() -> Vec<String> {
    demos![
        "primitives",
        "draw_indexed",
        "draw_indexed_push_constants",

        // ..
    ]
}

/// Register plugin system functions
#[no_mangle]
pub fn get_system_ecs_demos(name: String, view_name: String) -> Option<SystemDescriptor> {
    match name.as_str() {
        // setup functions
        "setup_draw_indexed" => system_func![setup_draw_indexed],
        "setup_primitives" => system_func![setup_primitives],
        "setup_draw_indexed_push_constants" => system_func![setup_draw_indexed_push_constants],

        // render functions
        "render_meshes" => render_func![render_meshes, view_name],

        // I had to add this `std::hint::black_box`!
        _ => std::hint::black_box(None)
    }
}
```

I hope to use `#[derive()]` macros to reduce the need for the boilerplate code, but I haven’t really looked into it in much detail yet. I had to add `std::hint::black_box` around the `None` case in the `get_system_` functions. I am here getting away without calling these functions without a c-abi wrapper so that might be the reason. Everything is working for the time being but I am prepared to address this if need be.

## Pmfx

Another core engine feature I have been working on is `pmfx` which is a high level platform agnostic graphics API that builds on top of the lower level `gfx` API. The idea here is that the `gfx` backends are fairly dumb wrapper API’s and `pmfx` can bring that low level functionality together in a way which is shared amongst different platforms, `pmfx` is also a data driven rendering system where render pipelines, passes, views, and graphs can be specified in [jsn](https://github.com/polymonster/jsn) config files to make light work of configuring rendering. This is not new code and it’s something I have worked on and used in other code bases, but it is currently undergoing an overhaul to bring it more inline with modern graphics API architectures. The main [pmfx-shader](https://github.com/polymonster/pmfx-shader) repository contains the data side of all of this.

So how does it work? You can write regular `hlsl` shaders and then supply `pmfx` files, which are used to create `pipelines`, `views`, `textures` (and render targets) and more. `views` are like render passes but with a bit more detail, such as a function that can be dispatched into a render pass with a camera for example.

```text
textures: {
    main_colour: {
        ratio: {
            window: "main_window",
            scale: 1.0
        }
        format: "RGBA8n"
        usage: ["ShaderResource", "RenderTarget"]
        samples: 8
    }
    main_depth(main_colour): {
        format: "D24nS8u"
        usage: ["ShaderResource", "DepthStencil"]
        samples: 8
    }
}
views: {
    main_view: {
        render_target: [
            "main_colour"
        ]
        clear_colour: [0.45, 0.55, 0.60, 1.0]
        depth_stencil: [
            "main_depth"
        ]
        clear_depth: 1.0
        viewport: [0.0, 0.0, 1.0, 1.0, 0.0, 1.0]
        camera: "main_camera"
    }
    main_view_no_clear(main_view): {
        clear_colour: null
        clear_depth: null
    }
}
```

The `pmfx` config files supply useful defaults to minimise the amount of members that need initialising to setup render state, and `pmfx` can parse `hlsl` files with extra context provided through `pipelines` to generate shader reflection info, descriptor layouts, and more, which is yet to come.

```text
pipelines: {
    mesh_debug: {
        vs: vs_mesh
        ps: ps_checkerboard
        push_constants: [
            "view_push_constants"
            "draw_push_constants"
        ]
        depth_stencil_state: depth_test_less
        raster_state: cull_back
        topology: "TriangleList"
    }
}
```

You can supply render graphs which are built at run-time with automatic resource transitions and barriers inserted based on dependencies, this is still in early stages because my use cases are currently quite simple but in time I expect this to grow a lot more:

```text
render_graphs: {
    mesh_debug: {
        grid: {
            view: "main_view"
            pipelines: ["imdraw_3d"]
            function: "render_grid"
        }
        meshes: {
            view: "main_view_no_clear"
            pipelines: ["mesh_debug"]
            function: "render_meshes"
            depends_on: ["grid"]
        }
        wireframe: {
            view: "main_view_no_clear"
            pipelines: ["wireframe_overlay"]
            function: "render_meshes"
            depends_on: ["meshes", "grid"]
        }
    }
}
```

You can take a look at a simple example of [pmfx](https://github.com/polymonster/hotline-data/blob/master/src/shaders) supplied with the hotline repository. Based on this file a reflection [info file](https://github.com/polymonster/pmfx-shader/blob/master/examples/outputs/v2_info.json) is generated, as well as recompiling the `hlsl` source into byte code with `DXC`. You can supply compile time flags that are evaluated and will generate shader permutations. Shaders which share the same source code, even though the permutation flags may differ, are hashed and re-used so as few as possible shaders are generated and compiled. `pmfx` also carefully tracks all shaders and render states so only minimal changes get reloaded.

### Pmfx Rust

The Rust side of `pmfx` uses [serde](https://docs.rs/serde/latest/serde/) to serialise and deserialise json into `hotline_rs::gfx` structures so they can be passed straight to the `gfx` API. `pmfx` tracks source files that are dependencies to build shaders or `pmfx` render configs and then re-builds are triggered when changes are detected.

All of the render config states and objects have hashes exported along with them and these hashes can be used to check for changes against live resources in use inside the `client`. Only changed resources get re-compiled and reloaded. With checks to minimise shader rebuilds and checks on reloads this will hopefully mitigate compilation costs where combinatorial explosion can occur due to having many shader permutations.

The `pmfx` API can be used to load `pipelines` and `render_graphs` and then those resources can be found by name. Ownership of the resources remains with `pmfx` itself and render systems can borrow the resources for a short time on the stack to pass them into command buffers.

```rust
// load and create resources
let pmfx_bindless = asset_path.join("data/shaders/bindless");
pmfx.load(pmfx_bindless.to_str().unwrap())?;
pmfx.create_pipeline(&dev, "compute_rw", swap_chain.get_backbuffer_pass())?;
pmfx.create_pipeline(&dev, "bindless", swap_chain.get_backbuffer_pass())?;

// borrow resources (we need to get a pipeline built for a compatible render pass)
let fmt = swap_chain.get_backbuffer_pass().get_format_hash();
let pso_pmfx = pmfx.get_render_pipeline_for_format("bindless", fmt).unwrap();
let pso_compute = pmfx.get_compute_pipeline("compute_rw").unwrap();

// use resource in command buffers
cmdbuffer.set_compute_pipeline(&pso_compute);

// ..

cmdbuffer.set_render_pipeline(&pso_pmfx);
```

Views are a `pmfx` feature that start to lean into the `bevy_ecs` which contains entities such as `cameras` and `meshes` that can be used to render world views. A view contains a command buffer that can be generated each frame, they also have a camera (view constants) that can be bound for the pass and a render pass to render into. This is all passed to a `bevy_ecs` system function, which is dispatched on the CPU concurrently with any other render systems. Each view has its own command buffer and the jobs are read-only (aside from writing the command buffer), so they can be safely dispatched on different threads at the same time. You can build command buffers and make draw calls like this:

```rust
#[no_mangle]
pub fn render_meshes(
    pmfx: &bevy_ecs::prelude::Res<PmfxRes>,
    view: &pmfx::View<gfx_platform::Device>,
    mesh_draw_query: bevy_ecs::prelude::Query<(&WorldMatrix, &MeshComponent)>) -> Result<(), hotline_rs::Error> {
        
    let pmfx = &pmfx.0;

    let fmt = view.pass.get_format_hash();
    let mesh_debug = pmfx.get_render_pipeline_for_format(&view.view_pipeline, fmt)?;
    let camera = pmfx.get_camera_constants(&view.camera)?;

    // setup pass
    view.cmd_buf.begin_render_pass(&view.pass);
    view.cmd_buf.set_viewport(&view.viewport);
    view.cmd_buf.set_scissor_rect(&view.scissor_rect);
    view.cmd_buf.set_render_pipeline(&mesh_debug);
    view.cmd_buf.push_constants(0, 16 * 3, 0, gfx::as_u8_slice(camera));

    for (world_matrix, mesh) in &mesh_draw_query {
        view.cmd_buf.push_constants(1, 16, 0, &world_matrix.0);
        view.cmd_buf.set_index_buffer(&mesh.0.ib);
        view.cmd_buf.set_vertex_buffer(&mesh.0.vb, 0);
        view.cmd_buf.draw_indexed_instanced(mesh.0.num_indices, 1, 0, 0, 0);
    }

    // end / transition / execute
    view.cmd_buf.end_render_pass();

    Ok(())
}
```

Resource transitions are an important part of modern graphics API and I am aiming to make this as smooth as possible. In the `pmfx` file you can provide `render_graphs`, which will automatically insert transitions based on state tracking. As mentioned above, all of the view render functions are dispatched concurrently on the CPU, but on the GPU they get executed in a specific order based on the render graph’s dependencies, with appropriate transitions inserted in between. This is still quite bare bones because I am not doing anything overly complicated yet, but I expect this aspect of `pmfx` to require a lot more attention as the project progresses. `pmfx` also provides the ability to insert resolves for MSAA resources which is like a special kind of transition.

## Challenges

This leg of the project has been by far the most challenging. I started to hit more difficulties with memory ownership than I had until now, and with the addition of `bevy_ecs` and multi-threading introduces different scenarios to handle. Mostly the difficulties are to do with memory ownership, borrowing and mutability. Sometimes the borrow checker can be brutal and small tasks to refactor code can send you down a wormhole you didn’t expect.

### Refactoring

Refactoring in general I have found more difficult at times in Rust than in any other language. I tend to start things quite quickly and get something working; this typically means creating separate objects on the stack inside `main` and then that means the data is a bit more favourable to avoid overlapping mutability / borrowing issues.

When performing what initially seems like a simple refactor to bring that code more inline with where you want it to be or where your mental model is, you can hit a load of borrow checker errors and it turns out to be a more challenging task than you thought due to the mutual exclusion property of mutable references or just trying to move something to a thread, maybe some types can’t be `Send` and then this means you have to re-think how your data is grouped together or how it is synchronised across threads.

### Ownership

Until this point I had mostly been dealing with objects on the stack that had been quite easy to either move or pass as reference through the call stack. Here are some examples of more complicated memory ownership:

#### Basic Lifetimes

I have a couple of places where I am using lifetimes, but have tried to steer away from them as much as possible. The current place I am using them is when passing `info` structures to create resources from a module backend.

```rust
/// Information to create a pipeline through `Device::create_render_pipeline`. where the shaders will be visible in the current stack
pub struct RenderPipelineInfo<'stack, D: Device> {
    /// Vertex Shader
    pub vs: Option<&'stack D::Shader>,
    /// Fragment Shader
    pub fs: Option<&'stack D::Shader>,

    // ..
}

/// The shader lifetime lasts long enough to pass to `Device::create_render_pipeline`
let vsc_info = gfx::ShaderInfo {
    shader_type: gfx::ShaderType::Vertex,
    compile_info: None
};
let vs = device.create_shader(&vsc_info, &vsc_data)?;

let psc_info = gfx::ShaderInfo {
    shader_type: gfx::ShaderType::Vertex,
    compile_info: None
};
let fs = device.create_shader(&psc_info, &psc_data)?;

let pso = device.create_render_pipeline(&gfx::RenderPipelineInfo {
    vs: Some(&vs),
    fs: Some(&fs),

    // ..
})?;
```

I have considered moving the objects that need lifetimes out of the structure and just passing them instead to the function so that lifetimes are not needed, but that means you lose the ability for `defaults` so I’m not sure. There is a situation to handle when passing a resource into a command buffer so that resource is to be used by the GPU as the resource can be dropped before it is used but I will cover that later.

#### Overlapping Mutability

Overlapping mutability has been tricky to get around at times. That is taking 2 mutable references to data that overlaps. It’s interesting to have to tackle this because it’s not something that you need to think about in C or C++, yet it is happening all the time, [load-hit-stores](https://en.wikipedia.org/wiki/Load-Hit-Store) occur when aliasing memory by two pointers as function arguments because they cannot be guaranteed to be different at compile time. Using `restrict` was something I used to do in the past when thinking about performance, but it’s not really something I think all that much about these days because the memory aliasing concept is quite abstracted. Rust forbids it and as a result you end up in difficult situations when grouping data in different ways.

It’s a natural instinct to want to bundle things together, maybe coming from a C background with context passing has got me leaning this way. But in `hotline` one of the more difficult scenarios I hit was when creating the `Client`. It felt to me natural that the `Client` could bundle together some common core functionality and be something passed around between plugins. But trouble arises when you need to borrow 2 members at the same time.

```rust
// call plugin ui functions
for plugin in &mut self.plugins {
    // ..

    self = ui_fn(self, plugin.instance, imgui_ctx); // cannot move self because it is borrowed as mutable (`for plugin in &mut self.plugins`)
}
```

I was able to work around this particular instance by moving the plugins into another vector, but also then I had to separate what members were part of the `Plugin` so that the `libs` could be accessed inside plugin functions to allow them to find functions to call.

```rust
// take the plugin mem so we can decouple the shared mutability between client and plugins
let mut plugins = std::mem::take(&mut self.plugins);

// call plugin ui functions
for plugin in &mut plugins {
    // ..

    self = ui_fn(self, plugin.instance, imgui_ctx); //now we can move self 
}
```

This illustrates to me how data ownership and grouping is quite a different beast in Rust to what I am used to.

#### Iterator Consumers

I found myself breaking apart algorithms and doing things like finding data that needs to be mutated in one pass, gathering the results then iterating over the results in a separate pass to separate the mutability.

```rust
// iterate over `pmfx_tracking` to check for changes, and reload data
for (_, tracking) in &mut self.pmfx_tracking {
    let mtime = fs::metadata(&tracking.filepath).unwrap().modified().unwrap();
    if mtime > tracking.modified_time {
        
        // perform a reload
        self.shaders.remove(shader); //!! this is not possible as `self` is already borrowed (`for (_, tracking) in &mut self.pmfx_tracking`)
        // ..

        // update modified time
        tracking.modified_time = fs::metadata(&tracking.filepath).unwrap().modified().unwrap();
    }
}
```

My instinct initially went for imperative style loops, but since then I started to adopt the iterator patterns using `filter`, `map`, `fold`, and `collect`. This means that creating a mutable collection and inserting into that collection can be replaced with an immutable collection.

```rust
// first collect paths that need reloading
let reload_paths = self.pmfx_tracking.iter_mut().filter(|(_, tracking)| {
    fs::metadata(&tracking.filepath).unwrap().modified().unwrap() > tracking.modified_time
}).map(|tracking| {
    tracking.1.filepath.to_string_lossy().to_string()
}).collect::<Vec<String>>();

// iterate over the paths we want to reload
for reload_filepath in reload_paths {
    if !reload_filepath.is_empty() {

        // repeat similarly inside, collecting resources that need updating first

        // find textures that need reloading
        let reload_textures = self.textures.iter().filter(|(k, v)| {
            self.pmfx.textures.get(*k).map_or_else(|| false, |src| {
                src.hash != v.0
            })
        }).map(|(k, _)| {
            k.to_string()
        }).collect::<HashSet<String>>();

        // ..

        // reloading outside of any iterator tied to self (self here is mutable)
        self.recreate_textures(device, &reload_textures);
    }
}
```

I have started to think this way a bit more, but it’s a little alien to me when I can achieve the same thing with a simple loop.

#### Moves

For the plugin libs and for `bevy_ecs` particularly I need to pass the hotline modules as resources to the `ecs` systems. In the end I settled on moving the entire hotline `Client` into the plugin functions, into the ecs `World` and back out again. The core hotline modules also need to be wrapped up to be a `Resource` for `bevy_ecs`.

This feels quite nice in a way, that in each plugin you have full ownership of `hotline` and can do what you like, which makes it possible to do things like asynchronous system updates through bevy `Scheduler` and let that have full control.

```rust
// move hotline resource into world
self.world.insert_resource(session_info);
self.world.insert_resource(DeviceRes(client.device));
self.world.insert_resource(AppRes(client.app));
self.world.insert_resource(MainWindowRes(client.main_window));
self.world.insert_resource(PmfxRes(client.pmfx));
self.world.insert_resource(ImDrawRes(client.imdraw));
self.world.insert_resource(UserConfigRes(client.user_config));

// update systems
self.schedule.run(&mut self.world);

// move resources back out
client.device = self.world.remove_resource::<DeviceRes>().unwrap().0;
client.app = self.world.remove_resource::<AppRes>().unwrap().0;
client.main_window = self.world.remove_resource::<MainWindowRes>().unwrap().0;
client.pmfx = self.world.remove_resource::<PmfxRes>().unwrap().0;
client.imdraw = self.world.remove_resource::<ImDrawRes>().unwrap().0;
client.user_config = self.world.remove_resource::<UserConfigRes>().unwrap().0;
self.session_info = self.world.remove_resource::<SessionInfo>().unwrap();
```

It requires a small amount of hokey-cokey to do so, which is also a little strange but I kind of like it, I had to break out the different modules inside the client to avoid overlapping mutability when using the `SwapChain`, `Device` and `CmdBuf`.

#### Arcs

I am aware I could wrap everything in an `Arc` to get interior mutability and that might remove the need for the moves, but I decided to try and use them only where necessary. I am using `Arc` and `Mutex` anywhere inter-thread synchronisation is necessary. I quite like lockless data structures in C. I will take a look at [tokio](https://tokio.rs) when I get a chance  but for now I was going with a heavy handed approach in a few places just to get the program structured how I would like. I have this `Reloader` and `ReloadResponder` setup that watches files and flags when changes have occurred, triggers a rebuild and reloads, the `ReloadResponder` is also the only place using `dyn` dispatch, there is still more work to do in that area as I struggled with trying to achieve polymorphic behaviour that I would implement in C++.

Another place using an `Arc` is in `pmfx` because `Views` need to be mutable in render functions and they are located within a `HashMap`, so it’s not possible to borrow mutable from `pmfx` itself without interior mutability. This led me to go with an `Arc`, however a move should be viable because only a single render system will ever write to a single view, so here it does feel unnecessary to require an `Arc` but I also had to work with `bevy_ecs` systems here which forced my hand slightly. A `RefCell` might also be a better option in this scenario.

#### In-Flight GPU Resources

Within a multi-buffered GPU rendering system, while the CPU is building command buffers for the current frame, another previous frame is being executed concurrently on the GPU. This introduces issues where if we decide to `drop` a resource on the CPU side, it may still be in-use on the GPU and by dropping the resource this will cause a D3D12 validation error and can lead to a device removal. I encountered this issue first with textures used for videos in the `av` API, so I added a `destroy_texture` function that passes ownership of the texture to the GPU `Device` and a function `cleanup_resources` will check the resources are no longer in use before `dropping` them. This goes a little against Rust's memory model with the need to explicitly `drop` at the right time.

```rust
// swap the texture to None, and pass ownership of texture to the device. Where it will be cleaned up safely
let mut none_tex = None;
std::mem::swap(&mut none_tex, &mut self.texture);
if let Some(tex) = none_tex {
    device.destroy_texture(tex);
}
```

In some places where full reloads are taking place there is a useful function on a `SwapChain`, which can wait for the last submitted frame to complete on the GPU and then any `drops` happening will be guaranteed to be safe before any new frames are submitted.  

```rust
// check if we have any reloads available
if self.reloader.check_for_reload() == ReloadState::Available {
    // wait for last GPU frame so we can drop the resources
    swap_chain.wait_for_last_frame();
    self.reload(device);
    self.reloader.complete_reload();
}
```

This solution is much nicer because the `drop` can just happen naturally. It might not be possible or desired to perform this hard sync with the GPU, so in future I expect to have to use the `destroy` functions more (and add them for different resource types).

### Build Times / Linker Issues / Debugging

Build times are currently the biggest problem; a `plugin` takes around 6 seconds to build with a little extra to complete the reload, so live code editing does not feel hugely responsive. Reloading shaders or render configs is very fast though, so that balances it out a bit if you work across code and shaders, and is all the more reason to use more GPU driven techniques / compute. The build times in full debug builds are much slower, but because of an issue with more than 65535 symbols exported from a plugin, which is not supported by the MSVC toolchain, I am forced to switch to `01` optimization for debug and that has similar performance to release.

```text
= note: LINK : fatal error LNK1189: library limit of 65535 objects exceeded
```

#### Profiling Build Times

This [post](https://fasterthanli.me/articles/why-is-my-rust-build-so-slow) has lots of detailed info about build times. I have tried to profile the build with the `cargo build -Z timings` option but it is only available on the `nightly` channel. Switching to nightly made it possible to run with the flag but I couldn’t see any output `cargo-timings` files, I wonder if the `-Z timings` is not available on Windows? As a result I am shooting in the dark a little here. I have done some exploration to figure out what might work best.

#### Experimenting With Build Times

I tried to separate out the plugins over more libs so that the core hotline lib did not have to depend on `bevy_ecs`. This didn’t make much of a positive difference because it created the requirement for an additional plugin with shared code. This attempt ended up with 5 total build artefacts, which ended in 13 second build times.

- `hotline_rs.dll`
- `client.exe`
- `ecs_base.dll`
- `ecs.dll`
- `ecs_demos.dll`

Each library or executable that requires building adds a noticeable constant cost which seems like the link time. So reducing the amount of libs actually helped improve build times and adding more only increased the build time. I moved the `ecs_base` plugin into `hotline`, which reduces the number of build artefacts and brings me to 6 second build times. If I build a single lib and executable the build time is around 10 seconds, so the live building is an improvement, if still not where I would like it to be but maybe this can improve in time.

For an end user the desired result is that they would not need to modify the `client`, the `hotline` lib or the core `ecs` and only work inside a plugin such as `ecs_demos`. So this has about a 6 second build time, which is not too bad, however when working on the core engine itself care needs to be taken to make sure the plugins and the core libs are in sync so that means building more artefacts. Building from clean and switching between release and debug also added a cost, so keeping the number of libs down was the way to go.

#### Avoiding Unnecessary Builds

Due to the build times being fairly long I had to ensure that any builds were not being triggered erroneously when they did not need to. With shaders inside the main repository it causes `cargo build` to think that `hotline` needs rebuilding when shaders have changed, even though this should not affect any of the libs or the executables. This is particularly painful because if modifying a shader the client is able ro rebuild and reload the shaders and associated pipelines very quickly, however the next time code is modified in a plugin it causes the plugin to rebuild the main `lib` as well, causing the total build time to reach about 10 seconds. If the shaders are excluded from the package in `Cargo.toml` then the issue does not occur, but the data is necessary to ship to users.

Due to other constraints I ended up moving the data into a separate repository, which mitigates the issue of rebuilding the main library. For now the problem is kept at bay, but it is a bit more work to maintain changes in the data repository. I need to find a more long term solution to this. Even editing the `todo.txt` file I have inside the repository causes a cargo build to take a few seconds. I know you can exclude directories from the package, which resolves the issue, but also I would like these things to publish to crates.

#### Working in Plugin Environment

Debugging in general is more difficult in the `plugin` environment, if you are attached to the debugger `plugin` rebuilds will fail because the `.pdb` is locked. So sometimes it means resorting to `println!` debugging when you need to debug the hot reload process itself. 

I added support for serialisation of the basic program state, which is synchronised between release and debug builds. It keeps the camera position and the currently selected demos, so even from a full restart you are right back where you left off. This makes those times where something goes terribly wrong, or the times you need to edit the core engine, just a little bit easier.

For the convenience that having hot reloaded plugins aims to provide, developing in that environment is quite tricky, so currently it feels like having plugins is an extra burden to carry around. It would be nice to be able to switch between statically linked and dynamically linked plugins and that would also be a good option for a final packaged build of an application, where the hot reloading would not be required. This is something I will look into when I get a chance.

### Error Handling

I have spent quite a lot of time handling errors and propagating them in a way to allow the client to continue running gracefully should something go wrong. It’s quite easy to quickly get things working just using `unwrap` to panic if something is missing, fix the issue and leave the unwrap there. That’s how I tend to like working in other code bases; if something is missing, assert and then fix that before moving on. In certain situations, like a game for instance, missing data should not be present in a final build so having all of the code to gracefully handle it always felt like extra baggage. But in some situations like this code base and in tools, you need to allow things to go wrong and interactively resolve them.  

Luckily Rust is really good at error handling and it actively wants you to do so, even in cases where I would hit a panic for a missing shader or some other data which may have had a typo in or an incorrect path. When I was hitting the panic I would know exactly where and then returning `Result` from a function allows the use of `?`, which makes a significant improvement to the readability of the code by reducing the need to unwrap. Here's a bloated messy initial setup, partly down to no being sure how to handle errors in `bevy_ecs` systems.

```rust
#[no_mangle]
 pub fn render_meshes(
     pmfx: bevy_ecs::prelude::Res<PmfxRes>,
     view_name: String,
     mesh_draw_query: bevy_ecs::prelude::Query<(&WorldMatrix, &MeshComponent)>) {

    // this is just code needed to get gfx resources and unwrap them to use in command buffer generation
    let arc_view = pmfx.get_view(&view_name);
    if arc_view.is_none() {
        return;
    }
    let arc_view = arc_view.unwrap();
    let view = arc_view.lock().unwrap();

    let fmt = view.pass.get_format_hash();

    let mesh_debug = pmfx.get_render_pipeline_for_format("mesh_debug", fmt);
    if mesh_debug.is_none() {
        return;
    }
    let mesh_debug = mesh_debug.unwrap();

    let camera = pmfx.get_camera_constants(&view.camera);
    if camera.is_none() {
        return;
    }
    let camera = camera.unwrap();

    // ..
}
```

And with the proper result propagation it looks much better, I was also able to unwrap and pass the `View` into the function instead of fetching it by name because now I was calling the function from a closure which gives a bit more control.

```rust
#[no_mangle]
 pub fn render_meshes(
     pmfx: bevy_ecs::prelude::Res<PmfxRes>,
     view_name: String,
     mesh_draw_query: bevy_ecs::prelude::Query<(&WorldMatrix, &MeshComponent)>) -> {

    let fmt = view.pass.get_format_hash();
    let mesh_debug = pmfx.get_render_pipeline_for_format(&view.view_pipeline, fmt)?;
    let camera = pmfx.get_camera_constants(&view.camera)?;

    // ..
}
```

There are still lots of combinations and things to test when it comes to error handling so I have added some initial tests to try and catch things that might go wrong, but I foresee this as ongoing work and need to get in the habit of thinking of that earlier on instead of quickly getting something working and refactoring.

### What's Next?

I’m pretty happy with the overall program structure and also the stability has been great. I think that's a good affirmation that all of the hard work playing ball with the borrow checker pays off in the long run. I still think there's a lot to think about in terms of memory ownership, this is one area I’m not as certain about as anything I have worked on for a long time. Next up I will be starting to add lighting and shadows, firstly I need to add these concepts into the `ecs` and then plan to work on clustered lighting and virtual shadow maps. There are a few bits of `pmfx` I need to add and hookup to make that possible, but in general the graphics side of the engine is really coming along.

I posted about this both on [twitter](https://twitter.com/polymonster) and [mastodon](https://mastodon.gamedev.place/@polymonster), I was keen to move to mastodon but still finding much more engagement on twitter. Give me a follow if you’re interested and check out the [GitHub](https://github.com/polymonster/hotline) or [crates.io](https://crates.io/crates/hotline-rs) page.  
