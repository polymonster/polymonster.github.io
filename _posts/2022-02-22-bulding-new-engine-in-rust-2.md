---
title: 'Building a new graphics engine in Rust - Part 2: Graphics API frontend, Direct3D12 and some initial thoughts about Rust'
date: 2022-02-22 21:40:00
---

In my previous [post](https://www.polymonster.co.uk/blog/bulding-new-engine-in-rust) I gave a high level overview of a new graphics engine project I was undertaking. I have been working quite consistently since then in the evenings and at weekends fleshing out a graphics API frontend called `hotline::gfx::` and the first graphics backend `hotline::gfx::direct3d12` implemented with the assistance of [windows-rs](https://github.com/microsoft/windows-rs). 

I haven’t quite finished the full API yet, but I am fairly happy with how it has taken shape and I think there is enough there now for me to cover the details about the API, some issues I encountered, and some initial thoughts about Rust. Just a few hours here and there really does make a difference… check out the wall of green for 2022, almost at full occupancy!

![github](/images/posts/rust-gfx/wall-of-green.png)

This is my first time writing a bindless graphics API so there are a few new concepts I have had to deal with. I have written a Vulkan backend which emulated binding so I was already familiar with descriptor sets. I have worked with all of the console graphics APIs for a few generations, as well as mobile and desktop and I would say that Metal is up there as one of my favourites, so with the design of a new API frontend I have taken some inspiration from Metal but also tried to follow Direct3D12 fairly closely as well. I intend to add cross platform support with Vulkan, Metal and WebGPU in the future so having prior knowledge of other APIs is handy… I have yet to look at WebGPU in detail but I Imagine it will slot in fine with the others.

I suppose an important thing to note at this time is that the `gfx::` interface is supposed to be fairly low level, so I want to be able to expose as much of the underlying graphics APIs as possible but make it a little bit more user friendly and tailor it to my needs. I also intend on making a higher level interface that will abstract away the lower level details and provide a lightweight and simple way to rapidly iterate and develop graphics algorithms. I have something similar in my C++ engine. A data config system is used to build `views` and is quite powerful, where you can write config files like [this](https://github.com/polymonster/pmtech/blob/master/assets/configs/common.jsn), share and reuse setup code, and rely on default values to minimise the amount of code. I want to retain this sort of functionality, but take it even further and I want to ensure the low level interface is as lean and direct as possible.
 
## API Overview

I have published the [docs](https://polymonster.github.io/hotline/hotline/gfx/index.html) already so you can get an overview of how `gfx::` looks, thanks again `cargo doc` for making me add extra comments to make the documentation read nicely. For graphics coders everything should already make sense, but the extra details you can add in documentation are useful to clarify usage. Here is a rundown of how to use the interface:

First create a `gfx::Device`. You might have more than one GPU or may want to use more than 1 device at a time. In the future I would also like to be able to create multiple devices using different rendering backends for A/B testing and comparing performance so I have looked into both static and dynamic polymorphism in Rust, which I will go into a bit more detail about later.

```rust
let mut device = gfx_platform::Device::create(&gfx::DeviceInfo{
    render_target_heap_size: num_buffers,
    ..Default::default()
});
```

Once you have a `gfx::Device` you can use it to allocate resources such as `Buffer`, `Texture`, or `RenderPipeline` and all of the associated render states. I decided to separate buffer and texture here like Metal does and not go for the single resource route like in Direct3D12. The reason for this is that I really like how Metal handles render targets, textures and depth stencils by supplying flags when creating a texture, so I am doing the same here. You can supply [`TextureUsage`](https://polymonster.github.io/hotline/hotline/gfx/struct.TextureUsage.html) when creating a texture and [`BufferUsage`](https://polymonster.github.io/hotline/hotline/gfx/enum.BufferUsage.html) for a buffer, which will internally create the appropriate resource views for APIs that need them. A device also contains a few `gfx::Heap`’s that resources will be automatically allocated into. I made a simple allocator with a free list and I have also allowed provision for custom heaps to be created and those to be supplied when creating a resource. Heaps are one area I will likely revisit as time goes on but for now everything is working quite well. I just have one single shader resource view heap, which all textures and buffers go into and I haven’t encountered any issues so far, although I get the feeling Nvidia hardware might be more forgiving than AMD as I have not had a single GPU hang.

The `os::` API can be used to create an `os::Window`, and `gfx::Device` can be used to create and bind a `gfx::SwapChain` to a window so you can start drawing something with multi-buffering. You need a `gfx::CmdBuf` to set render states created with the device and make draw calls. The `CmdBuf` also needs to be linked with a `SwapChain`, which controls a fence so that we do not overwrite inflight command buffers. A swap chain can be created with any number of internal buffers, and the command buffer itself keeps a number of internal buffers that are swapped between each frame. Once a frame is complete, a `CmdBuf` can be executed on a `Device`. The `Device` contains a single command queue to which all buffers can be executed. I initially toyed with the idea of exposing a `CmdQueue`, but I am not really sure when multiple queues would be necessary so I left it for now. I intend `CmdBuf`to be thread safe and allow different threads to build up command buffers which are executed in a specified order at the end of each frame.

```rust
let swap_chain_info = gfx::SwapChainInfo {
    num_buffers: num_buffers as u32,
    format: gfx::Format::RGBA8n
};

let mut swap_chain = device.create_swap_chain(&swap_chain_info, &window);
let mut cmd = device.create_cmd_buf(2);

let vertices = [
    Vertex {
        position: [0.0, 0.25, 0.0],
        color: [1.0, 0.0, 0.0, 1.0],
    },
    Vertex {
        position: [0.25, -0.25, 0.0],
        color: [0.0, 1.0, 0.0, 1.0],
    },
    Vertex {
        position: [-0.25, -0.25, 0.0],
        color: [0.0, 0.0, 1.0, 1.0],
    },
];

let buffer_info = gfx::BufferInfo {
    usage: gfx::BufferUsage::Vertex,
    format: gfx::Format::Unknown,
    stride: std::mem::size_of::<Vertex>(),
    num_elements: 3,
};

let vertex_buffer = device.create_buffer(&buffer_info, Some(gfx::as_u8_slice(&vertices)))?;
```

All of the above is pretty straightforward, I skimmed over a couple of things which required a bit more thought and decision making. I decided to go all in with a `gfx::RenderPass` even though Direct3D12 has the flexibility to do `OMSetRenderTargets` and Vulkan now has [dynamic rendering](https://www.khronos.org/blog/streamlining-render-passes). Render pass can be a bit of a pain to set up, but Metal also requires a render pass so I decided to follow this pattern as all 3 API’s can fit quite easily. It is a bit more work to set it up and it may seem a little inflexible, but I think the higher level data driven graphics interface I will plug on top later will abstract this baggage away.

Another reason I thought a render pass type would be useful is the requirement of a `gfx::Pipeline` to take a pass when it is created. This is really the most painful part; Direct3D12 and Metal render pipelines require the formats and sample counts of render targets and depth stencils to be supplied when creating a pipeline. Vulkan requires a valid VkRenderPass, so having a `gfx::RenderPass` is useful in this case because it can be supplied to the `gfx::RenderPipelineInfo` and the formats and sample counts can be obtained from the pass. An important tid-bit here is that pipelines can be reused on different passes providing the formats and sample counts are the same.

Having to link passes to pipelines is still a tricky thing to deal with if you have an editor or something that is dynamically creating pipelines and render targets. I read this blog a while ago from [nikas gray machinery] where he talked about this constraint. He proposed a nice solution to dynamically create and cache pipelines per thread on the fly inside a render context. I intend to push the dynamic pipeline creation into the higher level interface. I would like to avoid writing this code multiple times for different graphics backends, so I am choosing here to make the low level platform specific implementations simple wrappers that are as dumb as possible.

Descriptor sets are probably the most confusing part of modern APIs… In Vulkan you create a [VkDescriptorSetLayout](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/man/html/vkCreateDescriptorSetLayout.html), which is the equivalent of a [ID3D12RootSignature](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nn-d3d12-id3d12rootsignature) in Direct3D12. Metal has [Argument Buffers](https://developer.apple.com/documentation/metal/buffers/about_argument_buffers), which look slightly different at a glance and I have yet to implement a bindless Metal renderer, so I will get to that eventually. These concepts are key to bindless rendering so I need to embrace them. After initial digging, it turns out Vulkan and Direct3D12 are pretty much the same. I settled on a `DescriptorLayout` which describes a layout of multiple `DescriptorBinding`’s (textures, constant buffers, samplers), `PushConstants` or static `Sampler`’s.

State management seems like everyone’s least favourite part of modern APIs and it is a crucial part that older drivers like Direct3D11 dealt with for us. Currently I have exposed a `TransitionBarrier` that can supply before and after `ResourceState` and it is down to the user to manually transition resources. When I get to working on the higher level API I will be adding automatic state tracking, but in the low level API I want to keep things as simple as possible so there will be no state tracking under the hood. With a data driven render graph, even in a multithreaded environment, it is possible to work out the resource states easily and automatically from data.

With all that, here is a quick snippet of some rendering code to set up and render a basic triangle, I am making good use of the `Default` trait to try and minimise the verbosity of pipeline setup:

```rust
let vs = device.create_shader(&vs_info, src.as_bytes())?;
let fs = device.create_shader(&fs_info, src.as_bytes())?;

let pso = device.create_render_pipeline(&gfx::RenderPipelineInfo {
    vs: Some(vs),
    fs: Some(fs),
    input_layout: vec![
        gfx::InputElementInfo {
            semantic: String::from("POSITION"),
            index: 0,
            format: gfx::Format::RGB32f,
            input_slot: 0,
            aligned_byte_offset: 0,
            input_slot_class: gfx::InputSlotClass::PerVertex,
            step_rate: 0,
        },
        gfx::InputElementInfo {
            semantic: String::from("COLOR"),
            index: 0,
            format: gfx::Format::RGBA32f,
            input_slot: 0,
            aligned_byte_offset: 12,
            input_slot_class: gfx::InputSlotClass::PerVertex,
            step_rate: 0,
        },
    ],
    descriptor_layout: gfx::DescriptorLayout::default(),
    raster_info: gfx::RasterInfo::default(),
    depth_stencil_info: gfx::DepthStencilInfo::default(),
    blend_info: gfx::BlendInfo {
        alpha_to_coverage_enabled: false,
        independent_blend_enabled: false,
        render_target: vec![
            gfx::RenderTargetBlendInfo::default()
        ]
    },
    topology: gfx::Topology::TriangleList,
    patch_index: 0,
    pass: swap_chain.get_backbuffer_pass()
})?;
```

I have a more fully featured sample program which implements all of the features of the `gfx::` API; it can be found [here](https://github.com/polymonster/hotline/blob/master/samples/hello_world/main.rs). Currently it’s a bit messy and everything is in one big sample. I initially started breaking down functionality in small samples like in my old [engine](https://github.com/polymonster/pmtech), but due to changes in the API it generated a lot of extra work. I currently only have a couple of [tests](https://github.com/polymonster/hotline/blob/master/tests/tests.rs) I can run and will be filling out more tests and samples once I’ve nailed down the final details. 

## Polymorphism

The first problem I ran into was how to make a polymorphic compile time abstraction API. In C++ there are a number of ways I would go about it, but a typical one is to use the linker to exclude a file and include a different file. I found [gfx-rs](https://github.com/gfx-rs/gfx) useful as a reference because most of the info about polymorphism I searched for was from basic examples and didn’t quite go into enough detail. The Rust compiler and borrow checker can be quite a tricky customer, and even with a lot of coding experience it was quite a humbling experience being completely ripped to shreds and banging my head against a brick wall, so having some example code to refer to helped me a lot.

Rust separates data (`struct`) and behaviours (`impl`). It doesn’t have both combined in a class like C++ does. Shared behaviour can be implemented via a `trait`, which acts like interfaces. I am using a `Device` trait that contains functions and types. In C++ you could think all of this as virtual. A function can take a `self` parameter, which is a member function and if there is no `self` parameter to a function, the function is like a static C++ class function.

```rust
pub trait Device: Sized + Any {
    type SwapChain: SwapChain<Self>;
    type Heap: Heap<Self>;
    type CmdBuf: CmdBuf<Self>;
    // ..
    fn create(info: &DeviceInfo) -> Self;
    fn create_heap(&self, info: &HeapInfo) -> Self::Heap;
    fn create_swap_chain(&mut self, info: &SwapChainInfo, window: &platform::Window) -> Self::SwapChain;
    fn create_cmd_buf(&self, num_buffers: u32) -> Self::CmdBuf;
    // ..
}
```

We can enforce an implementation to implement concrete types such as `type Buffer<Self>`, and when we call a function we can create a buffer and return a `-> Self::Buffer` where `Self` is the Device and the types belong to the Device.

Rust does not have namespaces but has the concept of modules, which broadly speaking can be thought of as the file structure containing the code. The `gfx::` API is in a file called `gfx.rs` and `direct3d12.rs` is inside a subfolder called `gfx` so that the qualified name can be used as `gfx::direct3d12::`. The `gfx::` module is in the directory above `direct3d12::`, so all the types defined in gfx.rs need to be qualified with `super`. `direct3d12::` implements the `super::Device` trait and concrete types for all of the types declared inside the device.

```rust
pub struct Device {
    _adapter: IDXGIAdapter1,
    adapter_info: super::AdapterInfo,
    dxgi_factory: IDXGIFactory4,
    device: ID3D12Device,
    command_allocator: ID3D12CommandAllocator,
    command_list: ID3D12GraphicsCommandList,
    command_queue: ID3D12CommandQueue,
    pix: Option<WinPixEventRuntime>,
    shader_heap: Heap,
    rtv_heap: Heap,
    dsv_heap: Heap,
}

impl super::Device for Device {
    type SwapChain = SwapChain;
    type CmdBuf = CmdBuf;
    type Heap = Heap;
    // ..
    fn create(info: &super::DeviceInfo) -> Device {
        // ..
    }
```

When creating a device a compile time switch can be used to select a particular backend on platforms which support it. All other types are created from a `Device` so we only need to select which `Device` to create at compile time.

```rust
#[cfg(target_os = "windows")]
use hotline::os::win32 as os_platform;
use hotline::gfx::d3d12 as gfx_platform;

fn main() {
    let mut device = gfx_platform::Device::create(/* .. */);
}
```

I also had a quick look into fully dynamic polymorphism; I thought it would be cool to select which graphics backend to use at compile time or even have 2 different backends to compare performance. This is all done through the `dyn` keyword and requires a `std::boxed::Box` to put the dynamic type in. This confused me at first; I really didn’t understand `Box` just because of its name, but eventually it made sense where the box is the thing you pass around, and then when you know what it is you get the thing out of the box - it’s kind of like a pointer really! There are other things which could be dynamic such as `std::rc::Rc` and  `std::arc::Arc`. I will be digging into these a bit more once I get the multithreaded `gfx::CmdBuf` generation going.  

## Adventures with windows-rs

I have been using the official Microsoft windows-rs crate and it has been really good so far. I encountered a few problems and asked questions on the GitHub issues tracker. I got quick responses, which is really good to know people are actively working and maintaining this. My first real programming experience was with Visual Basic 6 and Visual C++ and Win32, so it feels nice to come full circle and do my first real Rust coding on Windows too.

The first [issue](https://github.com/microsoft/windows-rs/issues/1410) I raised was an issue where I was unable to resize a swap chain. You can see my ramblings in the issue itself. At first I didn’t have a clue how ownership worked in Rust, let alone how it worked in conjunction with COM objects as well. After a quick response confirming that COM references are tied to `std::mem::Drop` I was able to dig into the issue and figure out the cause. When creating a `D3D12_RESOURCE_BARRIER` the [`D3D12_RESOURCE_TRANSITION_BARRIER`](https://microsoft.github.io/windows-docs-rs/doc/windows/Win32/Graphics/Direct3D12/union.D3D12_RESOURCE_BARRIER_0.html) is marked as `std::mem::ManuallyDrop`; each frame 2 transitions are required for a basic clear screen… first from present to render target and then from render target to present. Each buffer in the swap chain would leak 2 references each frame because they were not being manually dropped. I was able to see that the reference count when closing the application would grow larger for 2 objects if you left the application running longer. The reference leak is present in the basic Direct3D12 example supplied with windows-rs. It makes sense that the manually drop is present here because we don’t want a resource to be cleared up while it is inflight on the GPU. I added a `Vec` to track in flight barriers and once the fence is signalled on the command buffer `std::mem::ManuallyDrop` can be called on the buffers to release them.  

I also encountered an issue where the begin, end and set marker [events](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-beginevent) for `ID3D12CommandList` are not supposed to be used directly and they have to be used via a dll and header in C++ called [WinPixEventRuntime]. I asked on the windows-rs GitHub if it was something that could be added and again I received quick responses that in this case it was out of the scope of windows-rs. Instead I was able to use the ABI as suggested by using windows `LoadLibrary` and `GetProcAddress`. I made [gist](https://gist.github.com/polymonster/28896d761c3592bab48a94924c0d7406) for it because there were some difficulties I had figuring out how to pass a pointer to a COM object from a windows-rs type.

```rust
pub fn begin_event_on_command_list(&self, command_list: ID3D12GraphicsCommandList, color: u64, name: &str) {
    unsafe {
        let null_name = CString::new(name).unwrap();
        let fn_begin_event : BeginEventOnCommandList = self.begin_event;
        // here is the hairy part! transmute a cloned address of the ID3D12GraphicsCommandList.0 (anonymous union)
        let p_cmd_list = std::mem::transmute::<IUnknown, *const core::ffi::c_void>(command_list.0.clone());
        fn_begin_event(p_cmd_list, color, PSTR(null_name.as_ptr() as _));
    }
}
```

## Dangling Pointers

Most of the Direct3D12 calls required are `unsafe` so I was taking care to think about what unsafe code was doing. I let my guard down when filling out the input layout structure. At first when I ran the code everything compiled and ran totally fine. It was only until running tests I would experience intermittent crashes in the test code. I was unable to reproduce the issue in debug or release builds, but it happened fairly consistently when running tests. Eventually I discovered that I could debug the test executable and the Direct3D12 validation layer made it clear what the issue was. The semantic names supplied to the `D3D12_INPUT_ELEMENT_DESC` seemed to be garbage memory. I swept over the code and noticed a glaring mistake:

```rust
fn create_input_layout(layout: &super::InputLayout) -> D3D12_INPUT_LAYOUT_DESC {
    // oops! the vec will be out of scope when the function returns
    let mut d3d12_elems : Vec<D3D12_INPUT_ELEMENT_DESC> = Vec::new();
    for elem in layout {
        d3d12_elems.push(D3D12_INPUT_ELEMENT_DESC{
            SemanticName: PSTR(elem.semantic.as_ptr() as _),
            SemanticIndex: elem.index,
            Format: to_dxgi_format(&elem.format),
            InputSlot: elem.input_slot,
            AlignedByteOffset: elem.aligned_byte_offset,
            InputSlotClass: match elem.input_slot_class {
                super::InputSlotClass::PerVertex => D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,
                super::InputSlotClass::PerInstance => D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA
            },
            InstanceDataStepRate: elem.step_rate,
        })
    }
    D3D12_INPUT_LAYOUT_DESC {
        // taking a pointer to some memory that will be dropped!
        pInputElementDescs: d3d12_elems.as_mut_ptr(),
        NumElements: d3d12_elems.len() as u32
    }
}
```

The issue was two-fold because the pointers to the semantic names had been moved and I was taking a pointer to a vec of element descriptors that was also being dropped. In order to fix this I refactored the code appropriately and on reflection it makes complete sense. The main problem I had here was that the code I initially wrote did not require an unsafe block, I did not have any borrow checker errors to work around like I had to do in other areas of the code base, so I felt indestructible. It just let me take a pointer and leave it dangling with the vector going out of scope. What I didn’t know is that use of raw pointers in Rust is unsafe, it just doesn’t require an `unsafe` block. So lessons learned and more care to be taken in the future. The fix was to refactor the code so the pointers remain in scope for the lifetime that they are needed:

```rust
fn create_render_pipeline(&self, info: &super::RenderPipelineInfo<Device>) -> result::Result<RenderPipeline, super::Error> {
    
    //..

    // sematic names and input element are in scope for this function
    let semantics = null_terminate_semantics(&info.input_layout);
    let mut elems = Device::create_d3d12_input_element_desc(&info.input_layout, &semantics);
    let input_layout = D3D12_INPUT_LAYOUT_DESC {
        pInputElementDescs: elems.as_mut_ptr(),
        NumElements: elems.len() as u32,
    };

    // ..

    // pointers remain valid while the CreateGraphicsPipelineState call completes
    Ok(RenderPipeline {
        pso: unsafe { self.device.CreateGraphicsPipelineState(&desc)? },
        root_signature: root_signature,
        topology: to_d3d12_primitive_topology(info.topology, info.patch_index)
    })
}
```

I also encountered other similar undefined behaviour before I realised that Rust strings are not null terminated when passed as a raw pointer to C API’s, so to make the API user friendly I am passing rust `String` or `&str` in the frontend and null terminating in the backend:

```rust
unsafe {
    let nullt_entry_point = CString::new(compile_info.entry_point.clone())?;
}
```

## Options and Results

I have been using Options and Results quite a lot and just naturally found myself using them because it feels like the right way to go about things in Rust. I am actually quite surprised because I dislike this sort of thing in C++ and prefer quite basic error handling through asserts and like to keep my code lean and minimal. Unwrapping options and results can introduce quite a lot of extra semantics and bloat to the code so I am still trying to find what works best for me. I ended up with some crazy chains of `as_ref` unwraps like this:

```rust
let mut clear_depth = 0.0;
let mut depth_begin_type = D3D12_RENDER_PASS_BEGINNING_ACCESS_TYPE_PRESERVE;
if info.ds_clear.is_some() {
    let ds_clear = info.ds_clear.as_ref().unwrap();
    if ds_clear.depth.is_some() {
        depth_begin_type = D3D12_RENDER_PASS_BEGINNING_ACCESS_TYPE_CLEAR;
        clear_depth = *ds_clear.depth.as_ref().unwrap();
    }
} else if info.discard {
    depth_begin_type = D3D12_RENDER_PASS_BEGINNING_ACCESS_TYPE_DISCARD;
}
```

I like supplying `None` into an option; it’s similar to passing `nullptr`. I like checking `is_some()`, which is similar to checking `if(ptr)` but then having to unwrap, and `as_ref` just adds a bit more code. I would like to be able to unwrap the options using the `?`  operator, which I found very useful for handling errors but you are unable to use `?` on an `Option` inside a function that returns a result.

There are a lot of ways you can unwrap options with `map` and `and_then`, but I also found out that you can use match statements and they make things a bit more concise, dropping the need for the `as_refs`. The above code example ended up being refactored to the following which reads a bit easier and deals with both depth and stencil clears:

```rust
match &info.ds_clear {
    None => {
        if info.discard {
            depth_begin_type = D3D12_RENDER_PASS_BEGINNING_ACCESS_TYPE_DISCARD;
            stencil_begin_type = D3D12_RENDER_PASS_BEGINNING_ACCESS_TYPE_DISCARD;
        }
    },
    Some(ds_clear) => {
        match &ds_clear.depth {
            Some(depth) => {
                depth_begin_type = D3D12_RENDER_PASS_BEGINNING_ACCESS_TYPE_CLEAR;
                clear_depth = *depth
            }
            None => ()
        }
        match &ds_clear.stencil {
            Some(stencil) => {
                stencil_begin_type = D3D12_RENDER_PASS_BEGINNING_ACCESS_TYPE_CLEAR;
                clear_stencil = *stencil
            }
            None => ()
        }
    }
}
```

Next Steps

I have started to investigate implementing an [imgui-sys](https://github.com/imgui-rs/imgui-rs/tree/main/imgui-sys) rendering and platform backend using hotlines `os::` and `gfx::` API. I really want to have support for multiple “viewports”. Once I’m done with that I will be making some more tests and sample applications, isolating graphics API features and making them nice and clean and understandable for anyone who wants to use it, as by that time I think the API will be locked down.

If you are interested in this project and its progress you can follow me on [GitHub](https://github.com/polymonster) and [Twitter](https://twitter.com/polymonster), you can contact me on [Discord](https://discord.com/invite/3yjXwJ8wJC) and I am also happy to take contributors or work with people so if you want to get involved let me know!

