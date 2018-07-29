---
layout: default
--- 

# pmtech

pmtech is a lightweight, cross-platform, multithreaded 3D engine currently supporting Windows, MacOS, Linux and iOS with OpenGL3.1+, OpenGLES3+ and Direct3D11 renderers. Take a look at the [github](https://github.com/polymonster/pmtech) repository to see the source code.

A common question other coders ask me when I talk about developing my own engine from the ground up is why don't I just use unity or unreal engine. This introduction is to answer that question and to discuss my motivations behind the project.

I have worked now for around 10 years professionally as a programmer on low level code for proprietary engines, graphics, tools and build and content pipelines. So I am somewhat doing in my sparetime for fun what I also do for my day job, but I have found working on my own tech to be more liberating and enjoyable as I can take a more idealistic approach to the code I am developing.
 
Initially the project started to give myself a framework in which I could easily start new projects to prototype and test new ideas, after that it took the path of trying to address some of the frustrating issues I had encountered in my professional work.

I wanted to make the codebase as simple as possible. Having seen code bases that had grown and become extremely complex over time, I was always tasked with last minute optimisations which often led to the analysis that the engine needed to be restructured or redesigned (but we had to make do with some hacky optimisations as opposed to re-writing the engine). This was because a lot of the performance cost was in the way objects communicated with one another, deep call stacks and complex hierarchies cause cache misses and carry a cost to simply call functions and when drilling down into a function body there was often hardly any time spent doing arithmetic or data transformation.

For this reason I decided to take a more c-style data oriented approach and try to avoid using object oriented paradigms as much as possible, one of the most notable features of the code base is a data oriented component entity system which has powerful code re-use through component aggregation but is more cache friendly than an object oriented approach.

I wont go into to much more detail about object oriented vs data oriented approaches as there is already a lot written about it already but the data oriented approach naturally lends itself to performance because it is less bloated and more cache friendly and aims toward transforming large amounts of homogenous data instead focusing on individual objects. data oriented code also lends itself to be easily optimised with SIMD which I intend to use where possible to increase throughput even further. As an additional performance consideration multithreading is built into the core systems from the outset. The renderer, audio and physics get processed on dedicated threads and the user thread which contains the main scene and logic is also on a dedicated thread.

I wanted the project to be quick and easy to port, the project is split into 2 libraries called pen (engine) and put (toolkit). pen contains platform specific code and put contains only platform agnostic code, the idea behind this is that only pen needs to be ported and then all of the higher level stuff comes along for the ride. pen wraps up all platform specific code into a very small set of modules: renderer, os (window, input), threads, timer, memory and file system. All platforms share the header for each module which contains a simple procedural api and the platform specific details are hidden in a cpp. Files are included or excluded on a platform basis, use of ifdefs is kept to a minimum (because they can be messy and ugly). The amount of code required to port a platform is minimal and I am making use of posix of stdlib where applicable so there are even less platform specific files than there are platforms.

So thats the motivation behind it, simple minamilistic api's with a strong focus on data oriented code and performance.

## Getting Started

Starting a new project in pmtech is quick and easy, one of the key features I wanted in a codebase was the ability to create new projects which have all the shared functionality of the engine and toolkit, have the ability to maintain or modify the engine and toolkit but also give myself an environment where I can experiment new ideas without polluting the shared code base.

To make life easy [premake5](https://premake.github.io/) is used to generate projects and [pmtech/tools](https://github.com/polymonster/pmtech/tree/master/tools/premake) contains some lua scripts which make creating a new set of workspaces for visual studio, xcode or gnu make files a simple process:

```lua
dofile "pmtech/tools/premake/options.lua"
dofile "pmtech/tools/premake/globals.lua"
dofile "pmtech/tools/premake/app_template.lua"

-- Solution
solution "new_workspace"
	location ("build/" .. platform_dir ) 
	configurations { "Debug", "Release" }
	startproject "new_project"
	buildoptions { build_cmd }
	linkoptions { link_cmd }
	
-- Engine (pen) Project	
dofile "pmtech/pen/project.lua"

-- Toolkit (put) Project	
dofile "pmtech/put/project.lua"

create_app_example( "new_project", script_path() )
```

To get hooked into pmtech all you need to do is define an entry point and a struct with startup params, heres a quick example main.cpp:

```c++
pen::window_creation_params pen_window{
    1280,          // width
    720,           // height
    4,             // MSAA samples
    "scene_editor" // window title / process name
};

PEN_TRV pen::user_entry(void* params)
{
    // Unpack the params passed to the thread and signal to the engine it ok to proceed
    pen::job_thread_params* job_params    = (pen::job_thread_params*)params;
    pen::job*               p_thread_info = job_params->job_info;
    pen::thread_semaphore_signal(p_thread_info->p_sem_continue, 1);
    
    for(;;)
    {
        // Main Loop
        
        // Msg from the engine we want to terminate
        if (pen::thread_semaphore_try_wait(p_thread_info->p_sem_exit))
            break;
    }
    
    // Shutdown
    
    // Signal to the engine the thread has finished
    pen::thread_semaphore_signal(p_thread_info->p_sem_terminated, 1);
    return PEN_THREAD_OK;
}
```
With that small amount of code you get a small executable which uses minimal amount of system resources, dedicated render and audio threads and whole host of powerful features for games and 3D graphics dev.

The last thing to do is actually generate the projects which can be done via premake directly or through the pmtech build pipeline which will handle multiple platforms and targets:

```bash
python3 pmtech/tools/build.py -actions code -platform osx -ide gmake -toolset clang
# see -help for more info..
```

## Build Pipeline

pmtech has a build pipeline for generating code projects and data into application ready binary formats:
- Generate code projects (vs2015, vs2017, xcode, gmake).
- Convert .obj and .dae (collada) files to binary format.
- Compile hlsl shaders, convert hlsl to glsl.
- Generate shader reflection info in .json format to know vertex layouts, constant locations, texture locations.
- Compress texture files to dds using nvidia texture tools.
- Copy configs, fonts and other data.
- Generate file dependencies and timestsamps for hot reloading.

I chose to use python for most of the build pipeline, because I wanted a higher level scripting language for easy access to modify files paths and directories, access to lots of utility code and libraries such as json, string operations and file system operations makes life easy for the type of work involved. python also offers great cross platform support for all my target platforms so only one set of scripts is required with minor portability code (no more need for .sh and .bat files with duplicated code). 

The build script can be invoked as follows:

```bash
python3 pmtech/tools/build.py -help
```

```text
--------------------------------------------------------------------------------
pmtech build -------------------------------------------------------------------
--------------------------------------------------------------------------------
run with no arguments for prompted input
commandline arguments
	-all <build all>
	-actions <action, ...>
		code - generate projects and workspaces
		shaders - generate shaders and compile binaries
		models - make binary mesh and animation files
		textures - compress textures and generate mips
		audio - compress and convert audio to platorm format
		fonts - copy fonts to data directory
		configs - copy json configs to data directory
	-platform <osx, win32, ios, linux>
	-ide <xcode4, vs2015, v2017, gmake>
	-clean <clean build, bin and temp dirs>
	-renderer <dx11, opengl>
	-toolset <gcc, clang, msc>
--------------------------------------------------------------------------------
All Jobs Done (5ms)
```
When data is built a .json file is generated containing dependencies, later the engine uses this info to check if files need to be hot-reloaded. At build time the dependecies are used to determine if files need to be re-built, and also check if build data needs to be deleted if the source files no longer exists.

```json
{
    "dir": "bin/osx/data/fonts",
    "files": [
        {
            "data/fonts/fontawesome-webfont.ttf": [
                {
                    "name": "/Users/alex.dixon/dev/pmtech/examples/../assets/fonts/fontawesome-webfont.ttf",
                    "timestamp": 1514655469.0
                }
            ]
        },
        {
            "data/fonts/cousine-regular.ttf": [
                {
                    "name": "/Users/alex.dixon/dev/pmtech/examples/../assets/fonts/cousine-regular.ttf",
                    "timestamp": 1514655469.0
                }
            ]
        }
    ]
}
```

## Multithreaded Architecture

To take advantage of modern processors it is essential that any performance intensive applications make use of more than one cpu core. I wanted to build in some implicit multithreaded systems into pmtech from the outset so that programs will use multiple threads of cpu cores without any explicit multithreaded code required.  

pmtech uses a producer consumer thread model, the user thread can be seen as the brains of the application co-ordinating tasks which then get processed asynchronously meaning that the api or driver overhead of the lower level systems is decoupled from user thread and we get to distribute work to other cpu cores.

In order to implement these multithreaded systems I am using a simple wrapper api which contains two versions of each function, one which captures the arguments and stores them in a command buffer and a second which takes the function arguments and passes them to the system api (ie. Direct3D or OpenGl, Fmod or Bullet).  

I first used this strategy to decouple the performance cost of an OpenGL driver without having to change the interface. By including gl.h inside a namespace and defining wrapper functions for all OpenGL functions it is possible to store all arguments in a command buffer which then gets dispatched on another thread.

I will cover some examples of how this works for the rendering api but the audio and physics api's follow the same pattern and this strategy can be used to easily multithread an entire procedural api.

The renderer api closely shadows Direct3D11 primarily because this was the first rendering platform I implemented and it was new when I originally stated the project. A small example below shows some texture functions with duplicated in the direct:: namespace and the texture_creation_params struct, other gpu state follows this same pattern ie. depth stencil state, blend state, clear state and so on.

```c++
struct texture_creation_params
{
    u32   width;
    u32   height;
    s32   num_mips;
    u32   num_arrays;
    u32   format;
    u32   sample_count;
    u32   sample_quality;
    u32   usage;
    u32   bind_flags;
    u32   cpu_access_flags;
    u32   flags;
    void* data;
    u32   data_size;
    u32   block_size;
    u32   pixels_per_block; // pixels per block in each axis, bc is 4x4 blocks so pixels_per_block = 4 not 16
    u32   collection_type;
};

// textures
u32  renderer_create_texture(const texture_creation_params& tcp);
u32  renderer_create_sampler(const sampler_creation_params& scp);
void renderer_set_texture(u32 texture_index, u32 sampler_index, u32 resource_slot, u32 shader_type, u32 flags = 0);

namespace direct
{
    // textures
    void renderer_create_texture(const texture_creation_params& tcp, u32 resource_slot);
    void renderer_create_sampler(const sampler_creation_params& scp, u32 resource_slot);
    void renderer_set_texture(u32 texture_index, u32 sampler_index, u32 resource_slot, u32 shader_type, u32 flags = 0);   
}
```

The functions outside of the direct namespace simply store all parameters into a struct which gets pushed into a ring buffer, all memory buffers are copied at this point so that pointers on the stack are ok to pass to these functions. For any resources such as texture or blend states a u32 is returned which is used as a handle to the resource, so set_texture takes the u32 handle returned by create_texture.. at the time I wrote the code I thought this was a good solution and it has worked out ok but if I was to re-implement this system I would probably create stronger types for the handles just to catch any programming errors, for example texture_handle, buffer_handle, render_target_handle and so on.

```c++
u32 renderer_create_texture(const texture_creation_params& tcp)
{
    switch ((pen::texture_collection_type)tcp.collection_type)
    {
        case TEXTURE_COLLECTION_NONE:
        case TEXTURE_COLLECTION_CUBE:
        case TEXTURE_COLLECTION_VOLUME:
            break;
        default:
            PEN_ASSERT_MSG(0, "inavlid collection type");
            break;
    }
    cmd_buffer[put_pos].command_index = CMD_CREATE_TEXTURE;
    memory_cpy(&cmd_buffer[put_pos].create_texture, (void*)&tcp, sizeof(texture_creation_params));
    cmd_buffer[put_pos].create_texture.data = memory_alloc(tcp.data_size);
    if (tcp.data)
    {
        memory_cpy(cmd_buffer[put_pos].create_texture.data, tcp.data, tcp.data_size);
    }
    else
    {
        cmd_buffer[put_pos].create_texture.data = nullptr;
    }
    u32 resource_slot                 = slot_resources_get_next(&k_renderer_slot_resources);
    cmd_buffer[put_pos].resource_slot = resource_slot;
    INC_WRAP(put_pos);
    return resource_slot;
}
```

As mentioned in the introduction to pmtech, I wanted to maintain a more data oriented approach and try to steer clear from object oriented paradigms. The internal command buffer system would be something that might lead people to go down an OO route for instance you would have command base and then inherit from that to have create_texture_command. To implement this without using OO I am using a union of structs to store the commands:

```c++
struct deferred_cmd
{
    u32 command_index;
    u32 resource_slot;

    union {
        u32                              command_data_index;
        shader_load_params               shader_load;
        set_shader_cmd                   set_shader;
        input_layout_creation_params     create_input_layout;
        buffer_creation_params           create_buffer;
    ...
```

Each command has a command_index which is used to identify the command in a switch statement, the switch statement then passes the command buffer arguments to the equivalent functions which are defined within the direct:: namespace after the direct function is called any memory that was allocated during the command generation step is freed.

```c++
void exec_cmd(const deferred_cmd& cmd)
{
    switch (cmd.command_index)
    {
        case CMD_CLEAR:
            direct::renderer_clear(cmd.command_data_index);
            break;
        case CMD_PRESENT:
            direct::renderer_present();
            break;
        case CMD_LOAD_SHADER:
            direct::renderer_load_shader(cmd.shader_load, cmd.resource_slot);
            memory_free(cmd.shader_load.byte_code);
            memory_free(cmd.shader_load.so_decl_entries);
            break;
        case CMD_SET_SHADER:
            direct::renderer_set_shader(cmd.set_shader.shader_index, cmd.set_shader.
            break;
        case CMD_LINK_SHADER:
            direct::renderer_link_shader_program(cmd.link_params, cmd.resource_slot)
            for (u32 i = 0; i < cmd.link_params.num_constants; ++i)
                memory_free(cmd.link_params.constants[i].name);
            memory_free(cmd.link_params.constants);
            if (cmd.link_params.stream_out_names)
                for (u32 i = 0; i < cmd.link_params.num_stream_out_names; ++i)
                    memory_free(cmd.link_params.stream_out_names[i]);
            memory_free(cmd.link_params.stream_out_names);
            break;
    ...
```

The user thread can call the non-direct functions to build up the command buffer. In the meantime the render thread will wait sleeping on a semaphore until another thread calls renderer_consume_command_buffer.

```c++
// clear screen
pen::viewport vp = {0.0f, 0.0f, (f32)pen_window.width, (f32)pen_window.he
pen::renderer_set_viewport(vp);
pen::renderer_set_rasterizer_state(raster_state);
pen::renderer_set_scissor_rect(rect{vp.x, vp.y, vp.width, vp.height});
pen::renderer_set_targets(PEN_BACK_BUFFER_COLOUR, PEN_BACK_BUFFER_DEPTH);
pen::renderer_clear(clear_state);
// bind vertex layout
pen::renderer_set_input_layout(input_layout);
// bind vertex buffer
u32 stride = sizeof(vertex);
pen::renderer_set_vertex_buffer(vertex_buffer, 0, stride, 0);
// bind shaders
pen::renderer_set_shader(vertex_shader, PEN_SHADER_TYPE_VS);
pen::renderer_set_shader(pixel_shader, PEN_SHADER_TYPE_PS);
// draw
pen::renderer_draw(3, 0, PEN_PT_TRIANGLELIST);
// present
pen::renderer_present();
pen::renderer_consume_cmd_buffer();
```

The ring buffer which contains the command buffer contains a put_pos and a get_pos, after each time a command is added or executed this value is incremented. To save cpu cycles on wrapping the command buffer contains a power of 2 mnumber of commands and a simple bitwise and is used to mask the value back to 0 when it wraps:

```c++
#define MAX_COMMANDS (1 << 18)
#define INC_WRAP(V)                  
    V = (V + 1) & (MAX_COMMANDS - 1);
```

The render thread waits on a semaphore until a user thread tells it to consume the command buffer, it takes the current put_pos of the command buffer before singalling the user thread can continue, this is so that partial commands are not executed if they have been generated which the renderer is executing commands. The render thread will continually execute commands until it reaches the end_pos and then it will wait until renderer_consume_command_buffer is called again.

```c++
if (thread_semaphore_try_wait(p_consume_semaphore))
{
    // put_pos might change on the producer thread.
    u32 end_pos = put_pos;
    // need more commands
    PEN_ASSERT(commands_this_frame < MAX_COMMANDS);
    thread_semaphore_signal(p_continue_semaphore, 1);
    // some api's need to set the current context on the caller thread.
    direct::renderer_make_context_current();
    while (get_pos != end_pos)
    {
        exec_cmd(cmd_buffer[get_pos]);
        INC_WRAP(get_pos);
    }
    commands_this_frame = 0;
    return true;
}
return false;
```

Up until now all the examples show data going one-way, the user thread makes commands and the render thread executes them. The render api provides a function to read back data, it takes a struct with some parameters and a call back function which will get the data when it's ready:

```c++
pen::resource_read_back_params rrbp;
rrbp.block_size         = 4;
rrbp.row_pitch          = volume_dim * rrbp.block_size
rrbp.depth_pitch        = volume_dim * rrbp.row_pitch;
rrbp.data_size          = rrbp.depth_pitch;
rrbp.resource_index     = rt->handle;
rrbp.format             = PEN_TEX_FORMAT_BGRA8_UNORM;
rrbp.call_back_function = image_read_back;
pen::renderer_read_back_resource(rrbp);

void image_read_back(void* p_data, u32 row_pitch, u32 depth_pitch, u32 block_size)
{
    // user handles data read back themseleves in a thread safe way
}
```

The physics and audio api also require data to be read back from the user thread, internally this is handled with double buffers for data which the user thread requires, the buffers are swapped each time the command buffer is consumed at a safe time after all data writing has completed, the synchronisation of the buffers is done via an atomic so it is lockless. The physics and audio api's contain the following accessors which are thread safe:

```c++
// Audio Accessors
pen_error audio_channel_get_state(const u32 channel_index, audio_channel_state* state);
pen_error audio_channel_get_sound_file_info(const u32 sound_index, audio_sound_file_info* info);
pen_error audio_group_get_state(const u32 group_index, audio_group_state* state);
pen_error audio_dsp_get_spectrum(const u32 spectrum_dsp, audio_fft_spectrum* spectrum);
pen_error audio_dsp_get_three_band_eq(const u32 eq_dsp, audio_eq_state* eq_state);
pen_error audio_dsp_get_gain(const u32 dsp_index, f32* gain);

// Physics Accessors
mat4 get_rb_matrix(const u32& entity_index);
mat4 get_multirb_matrix(const u32& multi_index, const s32& link_index);
f32  get_multi_joint_pos(const u32& multi_index, const s32& link_index);
u32  get_hit_flags(u32 entity_index);

```

Currently each system has a dedicated thread, for now I was happy with this approach although in future having more control over the scheduling might be required, having a pool of threads with a work sharing or work stealing approach and breaking tasks into smaller more granular jobs can lead to more optimal cpu utilisation as cores or hardware threads idle time can be kept to a minimum. But for now all applications using pmtech get multithreaded system without the user having to give much thought to it.

## Component Entity System

The component entity system in pmtech is found [here](https://github.com/polymonster/pmtech/tree/master/put/include/ces), it is the beating heart of any 3D application. A scene is filled with entities which are made up of multiple components. Data for each entity is stored in structure of arrays with contiguous memory for all components so that cache misses are kept to a minimum when iterating through a scene. The component entity system updates positions of objects, heirarchicaly transforms nodes by their parent, udates bounding volumes, constant buffers and render the scene. The instancing sample shows how a large number of entities can be updated quickly by rotating their transform performed by quaternion multiplication, on an intel core i7 8th gen this takes only 0.5ms for 32k entities.

The components active in each entity are stored in the bit mask entities:

```c++
if (p_sn->entities[dst] & CMP_GEOMETRY)
	instantiate_model_cbuffer(scene, dst);
```

Below is a list of all the core components, a game or application can then extend this system creating it's own components and update behaviours:

```c++
// Components
cmp_array<u64>                 entities;
cmp_array<u64>                 state_flags;
cmp_array<hash_id>             id_name;
cmp_array<hash_id>             id_geometry;
cmp_array<hash_id>             id_material;
cmp_array<Str>                 names;
cmp_array<Str>                 geometry_names;
cmp_array<Str>                 material_names;
cmp_array<u32>                 parents;
cmp_array<cmp_transform>       transforms;
cmp_array<mat4>                local_matrices;
cmp_array<mat4>                world_matrices;
cmp_array<mat4>                offset_matrices;
cmp_array<mat4>                physics_matrices;
cmp_array<cmp_bounding_volume> bounding_volumes;
cmp_array<cmp_light>           lights;
cmp_array<u32>                 physics_handles;
cmp_array<cmp_master_instance> master_instances;
cmp_array<cmp_geometry>        geometries;
cmp_array<cmp_pre_skin>        pre_skin;
cmp_array<cmp_physics>         physics_data;
cmp_array<cmp_anim_controller> anim_controller;
cmp_array<u32>                 cbuffer;
cmp_array<cmp_draw_call>       draw_call_data;
cmp_array<free_node_list>      free_list;
cmp_array<cmp_material>        materials;
cmp_array<cmp_material_data>   material_data;
cmp_array<material_resource>   material_resources;
cmp_array<cmp_shadow>          shadows;
```

In addition to accessing the components in a literal manner ie. (scene->world_matrices[n]) the components can also be accessed in a generic manner to perform bulk operations such as copy, move, save, load or scene resize. Certain components such as geomerty, cbuffer, material and physics also require additional resources to be created (ie. vertex buffer, cbuffer, bullet physics instance). When performing any copy, save or load operations simple components are copied via a mem copy and then after that specialisation code is called to create geometery resources and so on. 

This generic behaviour is implemented through templates and polymorphism. Each cmp_array allocates an array of data and it also stores the size of the component, the scene provides a function to get a component by index:

```c++
template <typename T>
struct cmp_array
{
    u32 size = sizeof(T);
    T*  data = nullptr;

    pen_inline T& operator[](size_t index)
    {
	return data[index];
    }

    pen_inline const T& operator[](size_t index) const
    {
	return data[index];
    }
};

struct generic_cmp_array
{
    u32   size;
    void* data;

    pen_inline void* operator[](size_t index)
    {
	u8* d  = (u8*)data;
	u8* di = &d[index * size];
	return (void*)(di);
    }
};

// Access to component data in a generic way
pen_inline generic_cmp_array& get_component_array(u32 index)
{
	generic_cmp_array* begin = (generic_cmp_array*)this;
	return begin[index];
}
```

New nodes are added to a scene by using a free list to find exmpty slots in the component arrays, for speed new nodes can be append to the end of the arrays to guaruntee a contiguous node list. The component arrays can be resized if there is not enough space for new nodes, this is handled as a batch operation using the generic component array.

Hierachical transforms work by simply storing the index of the parent entity in parents component, all parents and children must be stored contiguously in the arrays with the parent first. When update is performed the parents will be transformed first and any children's world matrix is determined by it's local matrix multiplied by it's parents. In order to ensure that heirarchies are contiguous and in the correct order there are functions provided for parenting selection which will sort nodes accordingly. Moving / changing heirachies and parenting entites will cost more time but the common case of updating world matrix works simply by just multiplying matrixes on a linear array. 

The scene is rendered using the pmfx higher level rendering library which is data driven, pmfx is responsible for setting all the rendering state up for a view. A scene view can be described as a camera rendering a scene into a render target. The component entity system will frustum cull objects by AABB, bind entity constant buffers and make draw calls, handling skinning, instancing and stand alone draw calls.

## pmfx: Shaders

pmfx::shaders in pmtech build upon hlsl and add powerful additional features to make developing shaders seamlessly integrate into the work flow. All shaders can be written in hlsl shader model 4/5 and for glsl the shader code is generated as part of the shader build process. Texture and declarations differ slightly to hlsl:

```c++
declare_texture_samplers
{
	texture_2d( diffuse_texture, 0 );
	texture_2d( normal_texture, 1 );
	texture_2d( specular_texture, 2 );	

	texture_3d( sdf_volume, 14 );
	texture_2d( shadowmap_texture, 15 );
};
```

The second parameter passed to texture_xx functions is the register to bind it to (ie. register(t0) in hlsl). A .info file is exported with every shader, this contains information such as texture names and the associated register it is bound to in hlsl. Upon loading shaders the pmtech opengl implementation will bind textures and buffers to the corresponding register location to emulate the behaviour of d3d and hlsl.

All shaders must use constant buffers / uniform buffers as opposed to single uniforms. Registers are be specified for constant buffers in the usual hlsl way:

```c++
cbuffer per_pass_view : register(b0)
{
	float4x4 vp_matrix;
	float4x4 view_matrix;
	float4x4 view_matrix_inverse;
	
	float4 camera_view_pos;
	float4 camera_view_dir;
};
```

Certain registers are constant and always in use:  
cbuffer 0 = per_pass_view (view materix / camera info for the currently rendering view).   
cbuffer 1 = per_draw_call (object world matrix, inverse world matrix and 8 floats of custom data).  
cbuffer 3 = per_pass_lights (list of lights and a count).  
cbuffer 4 = per_pass_shadow (list of shadow map matrices).  
cbuffer 5 = per_pass_shadow_distance_fields (list of signed distance field volume matrices).  
cbuffer 7 = material_data (specified in technique constants).  
texture 14 = signed distance field texture.  
texture 15 = shadow map texture.  

Files can be included to share functionality, modular functions can be found in lighting.slib, skinning.slib, maths.slib and more. These add different lighting equations, attenuation functions, skinning utility functions and more. Multiple shaders can exisit within the same file, a json object is used to specify which vsmain and psmain to use and to generate a shader technique much like hlsl/fx techniques. Techniques can also specify tweakable constants which will go into cbuffer 7 and they are automatically enumerated in pmtech editor:

```json
"forward_lit_sdf_shadow":
{
	"vs": "vs_main",
	"ps": "ps_forward_lit",
	"defines": ["SDF_SHADOW"],

	"constants":
	{
		"albedo": { "type": "float4", "widget": "colour", "default": [1.0, 1.0, 1.0, 1.0] },
		"roughness": { "type": "float", "widget": "slider", "min": 0, "max": 1, "default": 0.5 },
		"reflectivity": { "type": "float", "widget": "slider", "min": 0, "max": 1, "default": 0.5 },
		"surface_offset": { "type": "float", "widget": "slider", "min": 0, "max": 1, "default": 0.3 }
	}
},
```

Uber shaders can be created by using a modified if statement, defines are placed into the "defines" key in the pmfx json block. All if: statements are converted to #if defined() before compilation, the reason for prefering if style statements as opposed to # is personal preference of finding the indentation and nesting of complex #if #endif statements to be harder to follow:

```c
if:(SDF_SHADOW)
{
	float s = sdf_shadow_trace()
	light_col *= smoothstep( 0.0, 0.1, s);
}
```


